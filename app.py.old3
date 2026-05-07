import streamlit as st
import requests
import pandas as pd
import numpy as np
from datetime import datetime, time, date
from zoneinfo import ZoneInfo
import time as t_module

IST = ZoneInfo("Asia/Kolkata")

# ─── PAGE CONFIG ───────────────────────────────────────────────────────────────
st.set_page_config(
    page_title="FnO Breakout Scanner",
    page_icon="📈",
    layout="wide",
    initial_sidebar_state="expanded"
)

# ─── LOAD SECRETS ──────────────────────────────────────────────────────────────
ACCESS_TOKEN      = st.secrets.get("UPSTOX_ACCESS_TOKEN", "DEMO")
SCAN_INTERVAL_SEC = int(st.secrets.get("SCAN_INTERVAL_SECONDS", 300))
VOL_THRESHOLD     = float(st.secrets.get("VOLUME_THRESHOLD", 1.5))
PCR_BULL_THRESH   = float(st.secrets.get("PCR_BULL_THRESHOLD", 1.0))
DEMO_MODE         = str(st.secrets.get("DEMO_MODE", "true")).lower() == "true"

# ─── CUSTOM CSS ────────────────────────────────────────────────────────────────
st.markdown("""
<style>
  @import url('https://api.fontshare.com/v2/css?f[]=satoshi@400,500,700&display=swap');
  :root {
    --bg: #0f1117; --surface: #1a1c23; --surface2: #20232d;
    --border: rgba(255,255,255,0.08); --text: #e2e8f0; --muted: #94a3b8;
    --faint: #475569; --green: #22c55e; --red: #ef4444; --gold: #f59e0b;
    --blue: #3b82f6; --teal: #14b8a6; --radius: 8px;
  }
  html, body, [class*="css"] { font-family: 'Satoshi', sans-serif !important; }
  .stApp { background: var(--bg); color: var(--text); }
  [data-testid="stSidebar"] { background: var(--surface) !important; border-right: 1px solid var(--border); }
  [data-testid="stSidebar"] .stMarkdown h2 { color: var(--teal); font-size: 12px; letter-spacing: 0.08em; text-transform: uppercase; font-weight: 700; }
  .metric-card { background: var(--surface2); border: 1px solid var(--border); border-radius: var(--radius); padding: 16px 20px; text-align: center; }
  .metric-card .label { color: var(--muted); font-size: 12px; font-weight: 500; letter-spacing: 0.05em; text-transform: uppercase; margin-bottom: 4px; }
  .metric-card .value { font-size: 28px; font-weight: 700; }
  .metric-card .value.green { color: var(--green); }
  .metric-card .value.red { color: var(--red); }
  .metric-card .value.gold { color: var(--gold); }
  .metric-card .value.blue { color: var(--blue); }
  .status-bar { background: var(--surface2); border: 1px solid var(--border); border-radius: var(--radius); padding: 10px 16px; display: flex; align-items: center; gap: 8px; font-size: 13px; color: var(--muted); margin-bottom: 16px; }
  .status-dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  .status-dot.active  { background: var(--green); box-shadow: 0 0 6px var(--green); }
  .status-dot.inactive{ background: var(--faint); }
  .status-dot.waiting { background: var(--gold);  box-shadow: 0 0 6px var(--gold); }
  .section-header { display: flex; align-items: center; gap: 10px; border-bottom: 1px solid var(--border); padding-bottom: 10px; margin-bottom: 16px; }
  .section-header .icon  { font-size: 18px; }
  .section-header .title { font-size: 15px; font-weight: 700; color: var(--text); }
  .section-header .count { background: var(--surface2); color: var(--muted); border: 1px solid var(--border); border-radius: 20px; padding: 2px 10px; font-size: 12px; font-weight: 600; }
  .countdown-bar { background: var(--surface2); border: 1px solid var(--border); border-radius: var(--radius); padding: 10px 16px; font-size: 13px; color: var(--muted); margin-bottom: 16px; display: flex; align-items: center; gap: 10px; }
  .countdown-val { color: var(--teal); font-weight: 700; font-size: 15px; font-variant-numeric: tabular-nums; }
  div[data-testid="stExpander"] { background: var(--surface2); border: 1px solid var(--border); border-radius: var(--radius); }
  .stButton > button { background: var(--teal) !important; color: #000 !important; font-weight: 700 !important; border: none !important; border-radius: var(--radius) !important; }
  .stDataFrame { border: 1px solid var(--border) !important; border-radius: var(--radius) !important; }
</style>
""", unsafe_allow_html=True)

# ─── CONSTANTS ─────────────────────────────────────────────────────────────────
UPSTOX_BASE  = "https://api.upstox.com/v2"
MARKET_OPEN  = time(9, 15)
SCANNER_START= time(9, 30)
MARKET_CLOSE = time(15, 30)

DEMO_FNO_STOCKS = [
    "RELIANCE","TCS","INFY","HDFCBANK","ICICIBANK","SBIN","AXISBANK","KOTAKBANK",
    "WIPRO","HCLTECH","BAJFINANCE","BHARTIARTL","ITC","LT","MARUTI","TITAN",
    "NESTLEIND","ULTRACEMCO","SUNPHARMA","ONGC","NTPC","POWERGRID","COALINDIA",
    "BPCL","IOC","HINDALCO","TATASTEEL","JSWSTEEL","VEDL","SAIL","ADANIENT",
    "ADANIPORTS","BAJAJFINSV","HDFCLIFE","SBILIFE","PIDILITIND","ASIANPAINT",
    "DABUR","MARICO","COLPAL","GODREJCP","BIOCON","DRREDDY","CIPLA","DIVISLAB",
    "APOLLOHOSP","FORTIS","LUPIN","AUROPHARMA","TATACONSUM","MCDOWELL-N"
]

# ─── UPSTOX API HELPERS ────────────────────────────────────────────────────────
def get_headers(token):
    return {"Authorization": f"Bearer {token}", "Accept": "application/json"}

@st.cache_data(ttl=3600)
def fetch_fno_instruments(token):
    try:
        import io, gzip, json
        url = "https://assets.upstox.com/market-quote/instruments/exchange/NSE.json.gz"
        resp = requests.get(url, timeout=15)
        with gzip.GzipFile(fileobj=io.BytesIO(resp.content)) as f:
            data = json.load(f)
        syms = sorted({
            item["underlying_symbol"]
            for item in data
            if item.get("segment") == "NSE_FO" and item.get("underlying_type") == "EQUITY"
        })
        return syms if syms else DEMO_FNO_STOCKS
    except:
        return DEMO_FNO_STOCKS

def fetch_candles(token, instrument_key):
    try:
        url = f"{UPSTOX_BASE}/historical-candle/intraday/{instrument_key}/5minute"
        resp = requests.get(url, headers=get_headers(token), timeout=10)
        data = resp.json()
        if data.get("status") == "success" and data.get("data", {}).get("candles"):
            candles = data["data"]["candles"]
            df = pd.DataFrame(candles, columns=["timestamp","open","high","low","close","volume","oi"])
            df["timestamp"] = pd.to_datetime(df["timestamp"]).dt.tz_convert("Asia/Kolkata")
            df = df.sort_values("timestamp").reset_index(drop=True)
            return df
    except:
        pass
    return pd.DataFrame()

def fetch_option_chain_oi(token, underlying_key):
    try:
        contract_url = f"{UPSTOX_BASE}/option/contract?instrument_key={underlying_key}"
        resp = requests.get(contract_url, headers=get_headers(token), timeout=10)
        data = resp.json()
        if data.get("status") == "success" and data.get("data"):
            nearest = sorted({c["expiry"] for c in data["data"]})[0]
            chain_resp = requests.get(
                f"{UPSTOX_BASE}/option/chain?instrument_key={underlying_key}&expiry_date={nearest}",
                headers=get_headers(token), timeout=10
            )
            rows = chain_resp.json().get("data", [])
            ce_oi = sum(r.get("call_options", {}).get("market_data", {}).get("oi", 0) for r in rows)
            pe_oi = sum(r.get("put_options",  {}).get("market_data", {}).get("oi", 0) for r in rows)
            pcr = pe_oi / ce_oi if ce_oi > 0 else 1.0
            return pcr, ce_oi, pe_oi
    except:
        pass
    return None, 0, 0

# ─── INDICATORS ────────────────────────────────────────────────────────────────
def calc_ema(series, period):
    return series.ewm(span=period, adjust=False).mean()

def get_first_15min_candle(df):
    today = df["timestamp"].dt.tz_convert("Asia/Kolkata").dt.date.max()
    today_df = df[df["timestamp"].dt.tz_convert("Asia/Kolkata").dt.date == today].head(3)
    if len(today_df) < 3:
        return None
    return {
        "high":  today_df["high"].max(),
        "low":   today_df["low"].min(),
        "open":  today_df.iloc[0]["open"],
        "close": today_df.iloc[-1]["close"],
    }

def check_conditions(df, pcr, vol_mult, pcr_bull):
    if df is None or len(df) < 30:
        return None
    df = df.copy()
    df["ema9"]  = calc_ema(df["close"], 9)
    df["ema21"] = calc_ema(df["close"], 21)
    avg_vol = df["volume"].iloc[-21:-1].mean() if len(df) > 21 else df["volume"].mean()
    last = df.iloc[-1]
    first15 = get_first_15min_candle(df)
    if not first15:
        return None

    cmp, ema9, ema21, vol = last["close"], last["ema9"], last["ema21"], last["volume"]
    vol_ratio = vol / avg_vol if avg_vol > 0 else 0
    pcr_val = pcr if pcr is not None else 1.0

    bo = dict(c1=cmp > first15["high"], c2=ema9 > ema21, c3=vol_ratio > vol_mult, c4=pcr_val > pcr_bull)
    bd = dict(c1=cmp < first15["low"],  c2=ema9 < ema21, c3=vol_ratio > vol_mult, c4=pcr_val < pcr_bull)

    return {
        "close": round(cmp, 2), "ema9": round(ema9, 2), "ema21": round(ema21, 2),
        "volume": int(vol), "avg_volume": int(avg_vol), "vol_ratio": round(vol_ratio, 2),
        "pcr": round(pcr_val, 3),
        "first15_high": round(first15["high"], 2), "first15_low": round(first15["low"], 2),
        "breakout":  all(bo.values()),
        "breakdown": all(bd.values()),
        "bo_c1": bo["c1"], "bo_c2": bo["c2"], "bo_c3": bo["c3"], "bo_c4": bo["c4"],
        "bd_c1": bd["c1"], "bd_c2": bd["c2"], "bd_c3": bd["c3"], "bd_c4": bd["c4"],
    }

# ─── MOCK DATA ─────────────────────────────────────────────────────────────────
def generate_mock_result(symbol, bucket, vol_mult, pcr_bull):
    rng  = np.random.default_rng(bucket + abs(hash(symbol)) % 2**31)
    base = rng.uniform(100, 3000)
    cmp  = round(base * rng.uniform(0.97, 1.04), 2)
    ema9 = round(cmp * rng.uniform(0.98, 1.02), 2)
    ema21= round(cmp * rng.uniform(0.97, 1.03), 2)
    f15h = round(base * 1.01, 2)
    f15l = round(base * 0.99, 2)
    avg_v= int(rng.integers(50000, 500000))
    vol  = int(avg_v * rng.uniform(0.5, 3.0))
    pcr  = round(rng.uniform(0.5, 2.0), 3)
    vr   = round(vol / avg_v, 2)

    bo = dict(c1=cmp>f15h, c2=ema9>ema21, c3=vr>vol_mult, c4=pcr>pcr_bull)
    bd = dict(c1=cmp<f15l, c2=ema9<ema21, c3=vr>vol_mult, c4=pcr<pcr_bull)
    return {
        "symbol": symbol, "close": cmp, "ema9": ema9, "ema21": ema21,
        "volume": vol, "avg_volume": avg_v, "vol_ratio": vr, "pcr": pcr,
        "first15_high": f15h, "first15_low": f15l,
        "breakout": all(bo.values()), "breakdown": all(bd.values()),
        "bo_c1": bo["c1"], "bo_c2": bo["c2"], "bo_c3": bo["c3"], "bo_c4": bo["c4"],
        "bd_c1": bd["c1"], "bd_c2": bd["c2"], "bd_c3": bd["c3"], "bd_c4": bd["c4"],
    }

# ─── SESSION STATE ─────────────────────────────────────────────────────────────
defaults = {
    "scan_results": [], "last_scan_time": None,
    "next_scan_time": None, "scan_count": 0
}
for k, v in defaults.items():
    if k not in st.session_state:
        st.session_state[k] = v

# ─── SCANNER LOGIC ─────────────────────────────────────────────────────────────
def run_scan():
    now = datetime.now(IST)
    symbols = DEMO_FNO_STOCKS if DEMO_MODE else fetch_fno_instruments(ACCESS_TOKEN)[:100]
    bucket  = int(now.timestamp()) // SCAN_INTERVAL_SEC
    results = []
    prog = st.progress(0, text="Initialising scan…")
    for i, sym in enumerate(symbols):
        prog.progress((i + 1) / len(symbols), text=f"Scanning {sym}…")
        if DEMO_MODE:
            r = generate_mock_result(sym, bucket, VOL_THRESHOLD, PCR_BULL_THRESH)
        else:
            df = fetch_candles(ACCESS_TOKEN, f"NSE_EQ|{sym}")
            pcr, _, _ = fetch_option_chain_oi(ACCESS_TOKEN, f"NSE_EQ|{sym}")
            cond = check_conditions(df, pcr, VOL_THRESHOLD, PCR_BULL_THRESH)
            if not cond:
                continue
            r = {"symbol": sym, **cond}
        results.append(r)
    prog.empty()
    st.session_state.scan_results  = results
    st.session_state.last_scan_time= now
    st.session_state.next_scan_time= datetime.fromtimestamp(now.timestamp() + SCAN_INTERVAL_SEC, tz=IST)
    st.session_state.scan_count   += 1

# ─── AUTO-SCAN ENGINE ──────────────────────────────────────────────────────────
now          = datetime.now(IST)
is_weekday   = now.weekday() < 5
in_window    = is_weekday and SCANNER_START <= now.time() <= MARKET_CLOSE
next_scan    = st.session_state.next_scan_time
should_scan  = in_window and (next_scan is None or now >= next_scan)

if should_scan:
    run_scan()

# Compute seconds until next scan for countdown display
if st.session_state.next_scan_time and in_window:
    secs_left = max(0, int((st.session_state.next_scan_time - now).total_seconds()))
else:
    secs_left = None

# ─── SIDEBAR ───────────────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown("""
    <div style='display:flex;align-items:center;gap:10px;padding:12px 0 20px 0;
         border-bottom:1px solid rgba(255,255,255,0.08);margin-bottom:20px;'>
      <div style='font-size:24px;'>📊</div>
      <div>
        <div style='font-size:15px;font-weight:700;color:#e2e8f0;'>FnO Scanner</div>
        <div style='font-size:11px;color:#64748b;'>Breakout & Breakdown</div>
      </div>
    </div>""", unsafe_allow_html=True)

    st.markdown("## ⚙️ Active Config")
    mode_label = "🟡 Demo (simulated)" if DEMO_MODE else "🟢 Live (Upstox API)"
    st.markdown(f"**Mode:** {mode_label}")
    st.markdown(f"**Scan Interval:** every `{SCAN_INTERVAL_SEC}s`")
    st.markdown(f"**Volume Threshold:** `{VOL_THRESHOLD}×` avg")
    st.markdown(f"**PCR Bull Threshold:** `> {PCR_BULL_THRESH}`")
    st.caption("Edit `.streamlit/secrets.toml` to change these values.")

    st.markdown("---")
    st.markdown("## 📋 Market Status")
    market_open  = is_weekday and MARKET_OPEN <= now.time() <= MARKET_CLOSE
    st.markdown(f"**Market:** {'🟢 Open' if market_open else '🔴 Closed'}")
    st.markdown(f"**Scanner:** {'🟢 Running' if in_window else '⏸️ Waiting'}")
    st.markdown(f"**Time (IST):** `{now.strftime('%H:%M:%S')}`")
    st.markdown(f"**Date:** `{now.strftime('%a, %d %b %Y')}`")
    st.markdown(f"**Scans done:** `{st.session_state.scan_count}`")

    st.markdown("---")
    if st.button("🗑️ Clear Results", use_container_width=True):
        st.session_state.scan_results  = []
        st.session_state.last_scan_time= None
        st.session_state.next_scan_time= None
        st.session_state.scan_count    = 0
        st.rerun()

# ─── MAIN PAGE ─────────────────────────────────────────────────────────────────
st.markdown("""
<div style='padding:4px 0 20px 0;'>
  <h1 style='font-size:22px;font-weight:700;color:#e2e8f0;margin:0;'>
    FnO Breakout / Breakdown Scanner
  </h1>
  <p style='color:#64748b;font-size:13px;margin:4px 0 0 0;'>
    Auto-scans NSE FnO stocks · Mon–Fri · 9:30 AM – 3:30 PM IST
  </p>
</div>""", unsafe_allow_html=True)

# Status bar
dot  = "active" if in_window else ("waiting" if is_weekday else "inactive")
stat = (f"Scanner running — next scan in {secs_left}s" if secs_left is not None
        else ("Waiting for market window (9:30 AM IST)" if is_weekday else "Market closed — resumes Monday"))
last = st.session_state.last_scan_time
st.markdown(f"""
<div class="status-bar">
  <span class="status-dot {dot}"></span>
  <span>{stat}</span>
  <span style='margin-left:auto;color:#475569;font-size:12px;'>
    Last scan: {last.strftime('%H:%M:%S IST') if last else 'Not yet run'}
  </span>
</div>""", unsafe_allow_html=True)

# Auto-rerun so countdown ticks and next scan fires
if in_window:
    refresh_secs = min(secs_left, 30) if secs_left and secs_left > 0 else 30
    st.markdown(f'<meta http-equiv="refresh" content="{refresh_secs}">', unsafe_allow_html=True)

# ─── RESULTS ───────────────────────────────────────────────────────────────────
results = st.session_state.scan_results

if results:
    breakouts  = [r for r in results if r.get("breakout")]
    breakdowns = [r for r in results if r.get("breakdown")]
    near = [r for r in results if not r.get("breakout") and not r.get("breakdown") and (
        abs(r["close"] - r["first15_high"]) / r["first15_high"] < 0.005 or
        abs(r["close"] - r["first15_low"])  / r["first15_low"]  < 0.005
    )]

    # ── KPI row ──
    c1, c2, c3, c4 = st.columns(4)
    for col, label, val, cls in [
        (c1, "Total Scanned",  len(results),    "blue"),
        (c2, "🚀 Breakouts",   len(breakouts),  "green"),
        (c3, "📉 Breakdowns",  len(breakdowns), "red"),
        (c4, "👁️ Near Signal", len(near),       "gold"),
    ]:
        col.markdown(f"""<div class="metric-card">
          <div class="label">{label}</div>
          <div class="value {cls}">{val}</div>
        </div>""", unsafe_allow_html=True)

    st.markdown("<br>", unsafe_allow_html=True)

    # ── Helper to render table ──
    def render_table(rows, cols_src, cols_disp, bool_cols):
        df = pd.DataFrame(rows)[cols_src]
        df.columns = cols_disp
        for c in bool_cols:
            df[c] = df[c].map(lambda x: "✅" if x else "❌")
        st.dataframe(df, use_container_width=True, hide_index=True)

    # Breakouts
    st.markdown(f"""<div class="section-header">
      <span class="icon">🚀</span><span class="title">Breakout Stocks</span>
      <span class="count">{len(breakouts)}</span></div>""", unsafe_allow_html=True)
    if breakouts:
        render_table(breakouts,
            ["symbol","close","ema9","ema21","first15_high","vol_ratio","pcr","bo_c1","bo_c2","bo_c3","bo_c4"],
            ["Symbol","CMP","EMA 9","EMA 21","15m High","Vol×","PCR","Close>15H","EMA9>21","Vol✓","OI Bull"],
            ["Close>15H","EMA9>21","Vol✓","OI Bull"])
    else:
        st.markdown('<div style="color:#475569;padding:20px;text-align:center;background:#1a1c23;border-radius:8px;border:1px solid rgba(255,255,255,0.06);">No breakout stocks in this cycle.</div>', unsafe_allow_html=True)

    st.markdown("<br>", unsafe_allow_html=True)

    # Breakdowns
    st.markdown(f"""<div class="section-header">
      <span class="icon">📉</span><span class="title">Breakdown Stocks</span>
      <span class="count">{len(breakdowns)}</span></div>""", unsafe_allow_html=True)
    if breakdowns:
        render_table(breakdowns,
            ["symbol","close","ema9","ema21","first15_low","vol_ratio","pcr","bd_c1","bd_c2","bd_c3","bd_c4"],
            ["Symbol","CMP","EMA 9","EMA 21","15m Low","Vol×","PCR","Close<15L","EMA9<21","Vol✓","OI Bear"],
            ["Close<15L","EMA9<21","Vol✓","OI Bear"])
    else:
        st.markdown('<div style="color:#475569;padding:20px;text-align:center;background:#1a1c23;border-radius:8px;border:1px solid rgba(255,255,255,0.06);">No breakdown stocks in this cycle.</div>', unsafe_allow_html=True)

    # Near Signal
    if near:
        st.markdown("<br>", unsafe_allow_html=True)
        st.markdown(f"""<div class="section-header">
          <span class="icon">👁️</span><span class="title">Near Signal (within 0.5% of 15-min boundary)</span>
          <span class="count">{len(near)}</span></div>""", unsafe_allow_html=True)
        render_table(near,
            ["symbol","close","first15_high","first15_low","vol_ratio","pcr"],
            ["Symbol","CMP","15m High","15m Low","Vol×","PCR"], [])

    st.markdown("<br>", unsafe_allow_html=True)
    with st.expander("📘 Scanner Condition Logic"):
        st.markdown(f"""
**Breakout — All 4 must be TRUE:**
| # | Condition | Value |
|---|---|---|
| 1 | 5-min close **> 15-min candle High** | Price closed above ORB high |
| 2 | EMA 9 **> EMA 21** | Bullish short-term trend |
| 3 | Volume **> {VOL_THRESHOLD}×** 20-candle avg | Surge in buying interest |
| 4 | PCR **> {PCR_BULL_THRESH}** | More PE OI = bullish bias |

**Breakdown — All 4 must be TRUE:**
| # | Condition | Value |
|---|---|---|
| 1 | 5-min close **< 15-min candle Low** | Price broke below ORB low |
| 2 | EMA 9 **< EMA 21** | Bearish short-term trend |
| 3 | Volume **> {VOL_THRESHOLD}×** 20-candle avg | Surge in selling pressure |
| 4 | PCR **< {PCR_BULL_THRESH}** | More CE OI = bearish bias |

**PCR (Put-Call Ratio)** = Total PE OI ÷ Total CE OI (nearest expiry, all strikes).
        """)

else:
    if not in_window:
        st.markdown(f"""
        <div style='text-align:center;padding:60px 20px;'>
          <div style='font-size:48px;margin-bottom:16px;'>⏰</div>
          <div style='font-size:18px;font-weight:600;color:#e2e8f0;margin-bottom:8px;'>
            {'Waiting for 9:30 AM IST' if is_weekday else 'Market Closed'}
          </div>
          <div style='color:#64748b;font-size:14px;'>
            Scanner will start automatically at 9:30 AM IST on weekdays.
          </div>
        </div>""", unsafe_allow_html=True)
    else:
        st.markdown("""
        <div style='text-align:center;padding:60px 20px;'>
          <div style='font-size:48px;margin-bottom:16px;'>📡</div>
          <div style='font-size:18px;font-weight:600;color:#e2e8f0;margin-bottom:8px;'>Scanning…</div>
          <div style='color:#64748b;font-size:14px;'>First scan in progress.</div>
        </div>""", unsafe_allow_html=True)

st.markdown(f'<div style="color:#475569;font-size:11px;text-align:center;margin-top:32px;">FnO Scanner · {now.strftime("%d %b %Y %H:%M:%S IST")} · Scan #{st.session_state.scan_count}</div>', unsafe_allow_html=True)
