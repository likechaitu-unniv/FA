# GenOnce_AITradeBOT — Pre-Context for Code Changes

> **Purpose:** Read this file IN FULL before making any code change to this
> repo. It captures the architecture, every non-obvious design decision, all
> known active bugs and their fixes, and every invariant that must never be
> broken. Keep it updated whenever the codebase changes.

---

## 1. What this repo is

A **local, single-user, semi-automated intraday options-buying bot** for
NIFTY/BANKNIFTY/FINNIFTY/MIDCPNIFTY weekly options via Zerodha Kite Connect
**or** OpenAlgo (self-hosted broker-agnostic server).

- **BUYING only** — never sells to open a position.
- One AI/analysis call per Execute run, at the very start. The monitoring
  loop after entry never calls AI or does additional network analysis.
- `dry_run: false` in `settings.json` means **real money**. The default is
  `false` in this repo (changed from the template). This flag is
  intentionally NOT in the web UI — it must be flipped by hand.
- A force-exit safety net (`force_exit_time`, default 15:15) fires
  unconditionally on every monitoring tick regardless of what else is
  enabled.

---

## 2. File map

```
app.py              Flask + SocketIO web layer. No trading logic.
engine.py           Core orchestrator: builds universe, runs analysis,
                    enters positions, runs monitoring loop.
kite_client.py      KiteConnect wrapper (login, instruments, ticker, orders).
openalgo_client.py  OpenAlgo SDK wrapper — same interface as KiteClient.
option_signal.py    Pure-Python signal math (OI bias, VWAP, ORB, CPR, VIX,
                    probability, momentum, gap). NO I/O, NO network calls.
ai_sentiment.py     One AI call per run (Gemini or Claude). Normalises model
                    names. Returns POSITIVE/NEGATIVE/NEUTRAL + reason.
decision.py         Pure truth-table: decide_direction(). NO I/O at all.
settings.json       All user config. Read on every run; some fields
                    (SL/target/time-exit) re-read on every monitoring tick.
templates/index.html  Single-page dashboard (Jinja2).
static/dashboard.js   All frontend JS (SocketIO, charts, button logic).
static/style.css      Dashboard styles.
requirements.txt    kiteconnect, flask, flask-socketio, openalgo, requests,
                    pytest, simple-websocket.
```

---

## 3. Execution provider: Kite vs OpenAlgo

Controlled by `settings.json["execution_provider"]`:
- `"kite"` (default) → `KiteClient` — needs `kite_api_key` + `kite_api_secret`,
  browser OAuth login, KiteTicker WebSocket.
- `"openalgo"` → `OpenAlgoClient` — needs `openalgo_api_key` +
  `openalgo_host` (default `http://127.0.0.1:5000`), API key validation
  only (no browser), OpenAlgo WebSocket.

`engine.make_client(settings)` is the factory. **All engine functions use
the same interface regardless of provider.**

### Critical OpenAlgo differences vs Kite
| Concern | KiteClient | OpenAlgoClient |
|---|---|---|
| `instrument_token` | numeric int | tradingsymbol string |
| OI in WebSocket | present (FULL mode) | always 0 — must use REST `multiquotes()` |
| Spot index symbol | `"NIFTY 50"` on `NSE` | `"NIFTY"` on `NSE_INDEX` |
| VWAP `average_price` | real from tick | synthesised as `(open+ltp)/2` |
| Intraday history (index) | works directly | many plugins don't serve `NSE_INDEX` minute data — fall back to futures contract (see `get_intraday_history(fallback_token=...)`) |
| Login | browser OAuth | `funds()` API call only |
| Order entry | `place_market_order` | `optionsorder` / `optionsmultiorder` for entry; `placeorder` for square-off |

Detection: `is_openalgo = hasattr(kite_client, '_oa')` — used in
`gather_strike_oi`, `compute_positions_pnl`, `build_option_universe`,
`app.py` routes, etc.

---

## 4. Index → spot symbol mapping

```python
# Kite (engine.INDEX_SPOT_TRADINGSYMBOL)
NIFTY      → "NIFTY 50"       exchange NSE
BANKNIFTY  → "NIFTY BANK"     exchange NSE
FINNIFTY   → "NIFTY FIN SERVICE" exchange NSE
MIDCPNIFTY → "NIFTY MID SELECT"  exchange NSE

# OpenAlgo (engine.INDEX_SPOT_TRADINGSYMBOL_OA)
NIFTY      → "NIFTY"          exchange NSE_INDEX
BANKNIFTY  → "BANKNIFTY"      exchange NSE_INDEX
FINNIFTY   → "FINNIFTY"       exchange NSE_INDEX
MIDCPNIFTY → "MIDCPNIFTY"     exchange NSE_INDEX
```

`engine._resolve_spot_tradingsymbol(index, is_openalgo=False)` handles
the lookup. `build_option_universe` always calls this.

### India VIX symbol
`"INDIAVIX"` on `NSE_INDEX` for both providers. Retrieved via
`engine.get_india_vix()`.

---

## 5. Signal pipeline (one Execute run)

```
build_option_universe()       ← spot price, nearest expiry, ATM±5 strikes,
                                futures contract (VWAP proxy)
_setup_universe_and_ticker()  ← start WebSocket/ticker for all tokens
gather_strike_oi()            ← OI for all 22 options
                                Kite: from FULL-mode WS ticks
                                OpenAlgo: single REST multiquotes() call
get_oi_change()               ← today/yesterday OI ratio for walls + ATM
option_signal.compute_oi_walls()         ← nearest resistance/support strikes
option_signal.compute_oi_bias()          ← PCR (plain + proximity+OI-change weighted)
get_vwap()                    ← from futures tick/quote average_price,
                                fallback: minute candle VWAP
option_signal.compute_vwap_bias()
get_orb_bias()                ← first `orb_minutes` (default 15) of
                                1-min candles. OpenAlgo: falls back to
                                futures if index candles unavailable
option_signal.compute_orb_bias()
option_signal.combine_bias()  ← 2-of-3 technical vote (reference only)
fetch_daily_history()         ← shared: recent completed-day candles
get_gap_flag()                ← today open vs yesterday close vs avg range
get_india_vix()               ← REST quote for INDIAVIX
option_signal.compute_vix_expected_move()
option_signal.compute_avg_open_to_close_range()
blended_expected_move = (vix_move + historical_oc_range) / 2
get_atm_premiums()            ← live CE/PE LTP from tick or REST quote
option_signal.compute_momentum_bias()
option_signal.compute_volatility_confidence()
option_signal.compute_atm_oi_change_bias()
ai_sentiment.get_ai_sentiment()          ← ONE AI call, passes all quant context
option_signal.combine_probability_signals() ← 7-signal blend → upside%
decision.decide_direction()   ← CALL/PUT/BOTH/NO_TRADE
enter_positions() or NO_TRADE exit
monitor_and_exit()            ← loop: SL/target/time-exit/force-exit
```

---

## 6. Key invariants — never break these

1. **AI called exactly once per run**, before monitoring. The monitoring
   loop never calls AI or network analysis.
2. **Lot size always from live broker data**, never hardcoded.
3. **`force_exit_time` checked unconditionally** on every monitoring tick.
4. **`dry_run` is a hard gate** — if `True`, no real order ever reaches the
   broker. Never bypass it.
5. **`decision.decide_direction()` and `option_signal.*` are pure Python**
   — no I/O, no network calls, no side effects. Keep them that way.
6. **Square-off always checks live position qty** (via
   `get_actual_position_qty`) before placing a SELL — prevents accidental
   short if user already closed manually. Skipped in `dry_run`.
7. **Settings are re-read on every monitoring tick** for SL/target/
   time-exit (`engine._read_live_exit_settings`). `lots`, `index`, and API
   keys are fixed at entry time.
8. **`on_event` callbacks are fire-and-forget** — exceptions inside them
   must never propagate into trading logic.

---

## 7. settings.json — all fields

```jsonc
{
  "execution_provider": "kite" | "openalgo",
  "kite_api_key": "",
  "kite_api_secret": "",
  "openalgo_api_key": "",
  "openalgo_host": "http://127.0.0.1:5000",
  "proxy": {"enabled": false, "host": "", "port": "", "user": "", "pass": ""},
  "ai_provider": "gemini" | "claude",
  "gemini_api_key": "",
  "gemini_model": "gemini-3.5-flash",    // CURRENT working model (Oct 2026)
  "anthropic_api_key": "",
  "claude_model": "claude-sonnet-4-6",
  "index": "NIFTY" | "BANKNIFTY" | "FINNIFTY" | "MIDCPNIFTY",
  "lots": 5,
  "sl_enabled": true,
  "max_loss": 4000.0,                    // combined P&L SL in ₹
  "target_enabled": true,
  "target_profit": 10000.0,              // combined P&L target in ₹
  "pnl_mode": "COMBINED" | "PER_LEG",
  "time_exit_enabled": false,
  "time_exit": "10:30",
  "force_exit_time": "15:15",            // NOT editable from UI
  "pcr_threshold": 0.15,
  "range_lookback_days": 5,
  "gap_threshold_factor": 0.5,
  "dry_run": false,                      // NOT editable from UI
  "min_expected_move_points": 30,
  "call_probability_threshold": 0.55,
  "put_probability_threshold": 0.45,
  "custom_ai_note": "",
  "profit_margin_factor": 1.15,          // required_move = premium × this
  "momentum_min_fraction": 0.25,
  "momentum_max_fraction": 0.65,
  "oi_change_clamp": 0.5,
  "volatility_confidence_ratio_scale": 50.0,
  "volatility_confidence_orb_bonus": 15.0,
  "volatility_confidence_momentum_scale": 20.0,
  "volatility_confidence_oi_buildup_scale": 15.0,
  "volatility_confidence_cpr_narrow_scale": 15.0,
  "historical_range_lookback_days": 15,
  "atm_oi_change_threshold": 0.1,
  "orb_minutes": 15
}
```

**Fields NOT in the web UI (must be edited by hand in settings.json):**
`dry_run`, `force_exit_time`, `pnl_mode`, `pcr_threshold`,
`range_lookback_days`, `gap_threshold_factor`, `orb_minutes`,
`atm_oi_change_threshold`, all `volatility_confidence_*`.

---

## 8. app.py — web layer

- Flask on port **5051** (not 5050 as older versions; not 5000 which is the
  Kite redirect catcher).
- `STATE` dict (locked by `STATE_LOCK`) holds:
  `kite_client`, `logged_in`, `running`, `stop_event`,
  `loop_running`, `loop_stop_event`,
  `last_universe_event`, `last_decision_event`.
- `_VALID_INDICES = {"NIFTY","BANKNIFTY","FINNIFTY","MIDCPNIFTY"}` and
  `_ALL_INDICES = ["NIFTY","BANKNIFTY","FINNIFTY","MIDCPNIFTY"]`.
- `_parse_index_override(incoming)` returns the index string, `"ALL"`, or
  `None`.

### Routes
| Route | Method | Notes |
|---|---|---|
| `/` | GET | dashboard HTML (Jinja2) |
| `/api/settings` | POST | saves editable fields to settings.json |
| `/api/mode` | POST | toggle dry_run/LIVE badge |
| `/api/login` | POST | Kite OAuth browser flow |
| `/api/login/manual` | POST | manual token paste fallback |
| `/api/openalgo_login` | POST | OpenAlgo API key validation |
| `/api/execute` | POST | engine.run() in bg thread |
| `/api/analyze` | POST | engine.analyze_only() in bg thread |
| `/api/manual_trade` | POST | engine.manual_run(direction) |
| `/api/stop` | POST | sets stop_event |
| `/api/loop/start` | POST | start auto-loop bg thread |
| `/api/loop/stop` | POST | interrupt loop |
| `/api/status` | GET | logged_in, running, loop_running, restart flags |
| `/api/atm_preview` | GET | live ATM CALL/PUT LTPs for manual entry panel |
| `/api/spot_chart` | GET | today's 5-min candles for chart backfill |
| `/api/spot_ltp` | GET | single spot LTP poll |
| `/api/prompt_preview` | GET | current AI prompt text (read-only) |
| `/api/last_analysis` | GET | last universe_ready + decision events |

### Loop (`_run_loop`)
- mode: `"execute"` (calls `engine.run`) or `"analyze"` (calls
  `engine.analyze_only`).
- **Runs continuously until the user clicks Stop Loop** — never stops
  automatically.
- `index_override = "ALL"` → scans `_ALL_INDICES` in order within each
  iteration; moves to next index when NO_TRADE on the current one.

**Execution flow:**

```
loop starts
  │
  ├─ iteration N: engine.run() called
  │     ├─ NO_TRADE returned?
  │     │     └─ wait LOOP_INTERVAL_SEC (60s, interruptible)
  │     │         └─ → next iteration
  │     │
  │     └─ trade entered?
  │           engine.run() BLOCKS through entire trade lifecycle
  │           (entry → monitoring → SL/target/time-exit/force-exit → exit)
  │           position is FLAT when engine.run() returns
  │           └─ → next iteration IMMEDIATELY (no wait)
  │
  └─ repeat until Stop Loop clicked
```

- `LOOP_INTERVAL_SEC = 60` — only applied on NO_TRADE / failed iterations.
- After a completed trade, the next iteration starts with zero delay.
- Emits `loop_status {running, iteration, trade_count, mode, message}`.
- `trade_count` increments every time `engine.run()` returns with positions
  (i.e. a trade was entered and has now fully closed).
- The `stop_event` passed into `engine.run()` on each iteration is separate
  from the `loop_stop_event`. Clicking "Stop" (square-off button) stops the
  current trade; clicking "Stop Loop" stops the outer loop after the current
  `engine.run()` call finishes.

---

## 9. Gemini model — current working name

**`gemini-3.5-flash`** (stable, Oct 2026). Previous names that are now
dead/shut-down: `gemini-flash-latest`, `gemini-2.0-flash`, `gemini-2.5-flash`.

`ai_sentiment._normalise_gemini_model()` maps all legacy names to
`gemini-3.5-flash` automatically and logs a warning. If the model stops
working, update both `_normalise_gemini_model`'s alias map AND
`settings.json["gemini_model"]`.

---

## 10. Known issues resolved (do not re-introduce)

| Bug | Root cause | Fix location |
|---|---|---|
| Support/Resistance always `—` on OpenAlgo | OpenAlgo WS never sends OI | `engine.gather_strike_oi` → REST `multiquotes()` for OpenAlgo path |
| P&L always ₹0.00 on OpenAlgo | WS ticks miss; lookup by token string vs numeric | `engine.compute_positions_pnl` → REST batch fallback + dual-key tick lookup |
| ORB always NEUTRAL on OpenAlgo | NSE_INDEX minute history not served by broker plugin | `openalgo_client.get_intraday_history(fallback_token=futures)` |
| `today_open = 0.0` / wrong gap | `depth()` returns `open: 0.0` for index | Guard changed to `if not open_` (catches 0.0 and None) |
| VIX `ValueError` (`spot_price must be positive`) | `day_open` was `None` or `0.0` | `engine._run_analysis` falls back to `spot_price` when `day_open` falsy |
| AI sentiment HTTP 400 | Wrong model name `gemini-flash-latest` | `_normalise_gemini_model()` + settings.json updated |
| AI sentiment HTTP 404 | `gemini-2.0-flash` shut down Oct 2026 | Model updated to `gemini-3.5-flash` |
| FINNIFTY/MIDCPNIFTY `spot price = None` | Wrong OA symbol (`"NIFTY FIN SERVICE"`) | `INDEX_SPOT_TRADINGSYMBOL_OA` uses short names: `"FINNIFTY"`, `"MIDCPNIFTY"` |
| VWAP candle fallback crash | `kite_client.kite.historical_data()` called on OpenAlgoClient | Fallback uses `get_intraday_history(futures_tradingsymbol)` |
| Execute/Analyze disabled when ALL selected | `refreshControlState` is correct — ALL is loop-only | Hint bar `#allIndexHint` explains this to users |
| Stop button disabled during loop iterations | Was `!running \|\| loopRunning` | Changed to `!running` — Stop must always work |
| OpenAlgo `get_actual_position_qty` returns 0 / false "already exited" | `openposition(strategy=)` filters by strategy tag; mismatch returns 0 even though position is live | Three-layer check in `openalgo_client.get_actual_position_qty`: (1) `openposition(strategy="AlgoAI")`, (2) `positionbook()` full scan (strategy-agnostic), (3) `orderstatus()` of the entry order. Entry order IDs stored in `self._last_entry_order_ids` (populated by `place_market_order` BUY, `place_options_order`, `place_options_straddle`). Safe fallback returns `expected_qty` when all three layers fail so SELL is never silently skipped. |
| Loop stopped automatically after first trade | Old `_run_loop` broke out of the while-loop when `trade_entered=True` | Removed the `break`; after a completed trade the loop restarts immediately. Only `LOOP_INTERVAL_SEC` wait on NO_TRADE. Never stops until user clicks Stop Loop. |
| Loop shows `NO_TRADE` after a real trade completed | `trade_entered = bool(result.get("positions"))` — `positions` is always `[]` when `engine.run()` returns because `monitor_and_exit → square_off_all → positions.clear()` empties it before return | Changed to `result.get("direction") not in (None, "NO_TRADE")` — direction is always preserved |
| Status bar shows `| trades: 0` duplicate suffix | `dashboard.js` appended `| trades: N` on top of a server message that already contained the count | Removed the JS suffix; server message is shown directly |

---

## 11. dashboard.js state variables

```js
let loggedIn = false;
let running = false;       // engine is currently in a run/analysis
let loggingIn = false;
let loopRunning = false;   // auto-loop is active
```

`refreshControlState()` sets disabled states for all buttons based on the
combination of these four. Key rules:
- `ALL` selected in index dropdown → Execute + Analyze disabled (loop only).
- `loopRunning` → Execute, Analyze, Login, indexSelect, modeBadge disabled.
- `stopBtn.disabled = !running` — NOT blocked by `loopRunning`.
- `loopBtn.disabled = !loggedIn || running || !configured || loopRunning`.

Socket events emitted by server:
- `engine_status` `{running: bool}` — updates `running`.
- `loop_status` `{running, iteration, trade_count, mode, message}` — updates `loopRunning`. `trade_count` = number of trades that have fully completed in this loop session.
- `engine_event` `{type, ...}` — drives dashboard panels (decision, P&L, etc.).
- `login_status` `{status, source?, message?}` — drives login button state.
- `tick` `{instrument_token, last_price, ...}` — drives LTP/chart.

---

## 12. Client interface contract

Every method below must exist on both `KiteClient` and `OpenAlgoClient`
with identical signatures and return shapes:

```python
client.login(redirect_display_url=None)
client.complete_login(request_token)
client.get_nfo_option_chain(name, today=None)  → (expiry_date, [instrument_dict])
client.get_index_instrument(tradingsymbol, exchange="NSE")  → instrument_dict
client.get_nfo_futures_instrument(name, today=None)  → instrument_dict
client.get_quote([f"EXCH:SYM", ...])  → {key: {last_price, average_price, ohlc, oi, volume}}
client.get_daily_history(token, from_date, to_date, oi=False)  → [candle_dict]
client.get_intraday_history(token, from_dt, to_dt, interval="5minute", fallback_token=None)
client.start_ticker(tokens, token_meta=None, on_tick=None)
client.wait_for_ticker(timeout=10)  → bool
client.seconds_since_last_tick()  → float | None
client.stop_ticker()
client.get_tick(instrument_token)  → tick_dict | None
client.get_all_ticks()  → {token: tick_dict}
client.place_market_order(tradingsymbol, exchange, transaction_type, quantity)
client.get_actual_position_qty(tradingsymbol, exchange, product, expected_qty)
```

OpenAlgoClient additionally has:
```python
client.place_options_order(underlying, exchange_oa, expiry_str, option_type, qty)
client.place_options_straddle(underlying, exchange_oa, expiry_str, qty)
client._oa  # the raw OpenAlgo SDK api() object — used for is_openalgo detection
```

---

## 13. Adding new settings fields — checklist

1. Add the key + default to `settings.json`.
2. If editable from the UI: add the form field to `index.html`,
   add the key to `str_fields` / `int_fields` / `float_fields` in the
   `/api/settings` save endpoint in `app.py`,
   add the value to the `payload` in `settingsForm.submit` in `dashboard.js`.
3. If NOT editable from the UI: document it in section 7 above under
   "Fields NOT in the web UI".
4. Consume it in `engine.py` (or `option_signal.py` if it's a pure-math
   tunable).

---

## 14. Test files

```
test_option_signal.py  — pure math tests (no network, no broker, safe to run anytime)
test_decision.py       — decision truth-table tests (no network)
```

Run: `pytest test_option_signal.py test_decision.py -v`

---

## 15. File-change restart requirement

The server does **not** hot-reload `.py` files. After any `.py` edit:
1. Stop the existing `python app.py` process.
2. Run `python app.py` again.

Template (`templates/index.html`) and static files (`static/`) are served
fresh on every request — no restart needed for them.

---

## 16. Ports

| Port | Used for |
|---|---|
| 5051 | Flask dashboard (app.py, `PORT = 5051`) |
| 5000 | Kite OAuth redirect catcher (`kite_client.REDIRECT_PORT`) |

These two must never clash.

---

## 17. This repo vs AI-Plus-Algo-Trading/algo_ai

Both repos share the same architecture and most of the same code. Key
differences to be aware of:
- This repo (`gkbot/GenOnce_AITradeBOT`) is an **independent copy** — changes
  in one do not automatically appear in the other.
- Dashboard port: this repo = **5051**; `algo_ai` = **5050**.
- Both repos may diverge over time. Always check which repo you're editing.
- When porting a bug fix from one to the other, verify the target repo's
  version of the relevant file before applying — it may already have the
  fix, or the code structure may differ slightly.

---

*Last updated: October 2026 — loop redesign (continuous run-until-stopped), OpenAlgo 3-layer position check. Update this file whenever architecture, settings, known bugs, or invariants change.*
