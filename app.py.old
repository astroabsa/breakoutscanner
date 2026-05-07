import streamlit as st
import requests
import pandas as pd
import numpy as np
from datetime import datetime, time, date
import time as t_module
import threading

# ─── PAGE CONFIG ───────────────────────────────────────────────────────────────
st.set_page_config(
    page_title="FnO Breakout Scanner",
    page_icon="📈",
    layout="wide",
    initial_sidebar_state="expanded"
)

# ─── CUSTOM CSS ────────────────────────────────────────────────────────────────
st.markdown("""
<style>
  @import url('https://api.fontshare.com/v2/css?f[]=satoshi@400,500,700&display=swap');

  :root {
    --bg: #0f1117;
    --surface: #1a1c23;
    --surface2: #20232d;
    --border: rgba(255,255,255,0.08);
    --text: #e2e8f0;
    --muted: #94a3b8;
    --faint: #475569;
    --green: #22c55e;
    --red: #ef4444;
    --gold: #f59e0b;
    --blue: #3b82f6;
    --teal: #14b8a6;
    --radius: 8px;
  }

  html, body, [class*="css"] { font-family: 'Satoshi', sans-serif !important; }
  .stApp { background: var(--bg); color: var(--text); }
  
  /* Sidebar */
  [data-testid="stSidebar"] { background: var(--surface) !important; border-right: 1px solid var(--border); }
  [data-testid="stSidebar"] .stMarkdown h2 { color: var(--teal); font-size: 13px; letter-spacing: 0.08em; text-transform: uppercase; font-weight: 700; }

  /* Metric cards */
  .metric-card {
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 16px 20px;
    text-align: center;
  }
  .metric-card .label { color: var(--muted); font-size: 12px; font-weight: 500; letter-spacing: 0.05em; text-transform: uppercase; margin-bottom: 4px; }
  .metric-card .value { font-size: 28px; font-weight: 700; }
  .metric-card .value.green { color: var(--green); }
  .metric-card .value.red { color: var(--red); }
  .metric-card .value.gold { color: var(--gold); }
  .metric-card .value.blue { color: var(--blue); }

  /* Stock pill badge */
  .badge-breakout { background: rgba(34,197,94,0.15); color: var(--green); border: 1px solid rgba(34,197,94,0.3); border-radius: 4px; padding: 2px 8px; font-size: 11px; font-weight: 700; letter-spacing: 0.06em; }
  .badge-breakdown { background: rgba(239,68,68,0.15); color: var(--red); border: 1px solid rgba(239,68,68,0.3); border-radius: 4px; padding: 2px 8px; font-size: 11px; font-weight: 700; letter-spacing: 0.06em; }
  
  /* Table tweaks */
  .stDataFrame { border: 1px solid var(--border) !important; border-radius: var(--radius) !important; }
  
  /* Status bar */
  .status-bar {
    background: var(--surface2);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 10px 16px;
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 13px;
    color: var(--muted);
    margin-bottom: 16px;
  }
  .status-dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  .status-dot.active { background: var(--green); box-shadow: 0 0 6px var(--green); }
  .status-dot.inactive { background: var(--faint); }
  .status-dot.waiting { background: var(--gold); }

  /* Section headings */
  .section-header {
    display: flex; align-items: center; gap: 10px;
    border-bottom: 1px solid var(--border);
    padding-bottom: 10px;
    margin-bottom: 16px;
  }
  .section-header .icon { font-size: 18px; }
  .section-header .title { font-size: 15px; font-weight: 700; color: var(--text); }
  .section-header .count { background: var(--surface2); color: var(--muted); border: 1px solid var(--border); border-radius: 20px; padding: 2px 10px; font-size: 12px; font-weight: 600; }

  /* Condition chips */
  .cond-pass { color: var(--green); font-size: 13px; }
  .cond-fail { color: var(--red); font-size: 13px; }

  div[data-testid="stExpander"] { background: var(--surface2); border: 1px solid var(--border); border-radius: var(--radius); }
  
  .stButton > button {
    background: var(--teal) !important;
    color: #000 !important;
    font-weight: 700 !important;
    border: none !important;
    border-radius: var(--radius) !important;
  }
  .stButton > button:hover { filter: brightness(1.1); }

  /* Warning alert box */
  .market-closed-box {
    background: rgba(245,158,11,0.1);
    border: 1px solid rgba(245,158,11,0.4);
    border-radius: var(--radius);
    padding: 16px 20px;
    color: var(--gold);
    font-size: 14px;
    margin-bottom: 16px;
  }
</style>
""", unsafe_allow_html=True)

# ─── CONSTANTS ─────────────────────────────────────────────────────────────────
UPSTOX_BASE = "https://api.upstox.com/v2"
MARKET_OPEN = time(9, 15)
SCANNER_START = time(9, 30)
MARKET_CLOSE = time(15, 30)

# Popular FnO stocks for demo (replace with live API call)
DEMO_FNO_STOCKS = [
    "RELIANCE","TCS","INFY","HDFCBANK","ICICIBANK","SBIN","AXISBANK","KOTAKBANK",
    "WIPRO","HCLTECH","BAJFINANCE","BHARTIARTL","ITC","LT","MARUTI","TITAN",
    "NESTLEIND","ULTRACEMCO","SUNPHARMA","ONGC","NTPC","POWERGRID","COALINDIA",
    "BPCL","IOC","HINDALCO","TATASTEEL","JSWSTEEL","VEDL","SAIL","ADANIENT",
    "ADANIPORTS","BAJAJFINSV","HDFCLIFE","SBILIFE","PIDILITIND","ASIANPAINT",
    "DABUR","MARICO","COLPAL","GODREJCP","BIOCON","DRREDDY","CIPLA","DIVISLAB",
    "APOLLOHOSP","FORTIS","LUPIN","AUROPHARMA","TATACONSUM","MCDOWELL-N"
]

# ─── UPSTOX API ─────────────────────────────────────────────────────────────────
def get_headers(access_token):
    return {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    }

@st.cache_data(ttl=3600)
def fetch_fno_instruments(access_token):
    """Fetch all NSE FnO instruments from Upstox instrument master."""
    try:
        url = "https://assets.upstox.com/market-quote/instruments/exchange/NSE.json.gz"
        import io, gzip, json
        resp = requests.get(url, timeout=15)
        with gzip.GzipFile(fileobj=io.BytesIO(resp.content)) as f:
            data = json.load(f)
        fno_symbols = list({
            item["underlying_symbol"] 
            for item in data 
            if item.get("segment") == "NSE_FO" and item.get("underlying_type") == "EQUITY"
        })
        return sorted(fno_symbols) if fno_symbols else DEMO_FNO_STOCKS
    except Exception as e:
        return DEMO_FNO_STOCKS

def fetch_candles(access_token, instrument_key, interval="5minute", days=1):
    """Fetch OHLCV candles from Upstox historical/intraday API."""
    try:
        today = date.today().strftime("%Y-%m-%d")
        url = f"{UPSTOX_BASE}/historical-candle/intraday/{instrument_key}/{interval}"
        resp = requests.get(url, headers=get_headers(access_token), timeout=10)
        data = resp.json()
        if data.get("status") == "success" and data.get("data", {}).get("candles"):
            candles = data["data"]["candles"]
            df = pd.DataFrame(candles, columns=["timestamp","open","high","low","close","volume","oi"])
            df["timestamp"] = pd.to_datetime(df["timestamp"])
            df = df.sort_values("timestamp").reset_index(drop=True)
            return df
    except:
        pass
    return pd.DataFrame()

def fetch_market_quote(access_token, instrument_keys):
    """Fetch full market quotes for a list of instrument keys."""
    try:
        keys = ",".join(instrument_keys)
        url = f"{UPSTOX_BASE}/market-quote/quotes?instrument_key={keys}"
        resp = requests.get(url, headers=get_headers(access_token), timeout=10)
        data = resp.json()
        if data.get("status") == "success":
            return data.get("data", {})
    except:
        pass
    return {}

def fetch_option_chain_oi(access_token, underlying_key):
    """Fetch ATM option chain to compute PCR (OI-based bull/bear signal)."""
    try:
        url = f"{UPSTOX_BASE}/option/chain?instrument_key={underlying_key}&expiry_date="
        # Get nearest expiry from contracts
        contract_url = f"{UPSTOX_BASE}/option/contract?instrument_key={underlying_key}"
        resp = requests.get(contract_url, headers=get_headers(access_token), timeout=10)
        data = resp.json()
        if data.get("status") == "success" and data.get("data"):
            expiries = sorted(set(c["expiry"] for c in data["data"]))
            nearest = expiries[0]
            chain_url = f"{UPSTOX_BASE}/option/chain?instrument_key={underlying_key}&expiry_date={nearest}"
            chain_resp = requests.get(chain_url, headers=get_headers(access_token), timeout=10)
            chain_data = chain_resp.json()
            if chain_data.get("status") == "success":
                rows = chain_data.get("data", [])
                ce_oi = sum(r.get("call_options", {}).get("market_data", {}).get("oi", 0) for r in rows)
                pe_oi = sum(r.get("put_options", {}).get("market_data", {}).get("oi", 0) for r in rows)
                pcr = pe_oi / ce_oi if ce_oi > 0 else 1.0
                return pcr, ce_oi, pe_oi
    except:
        pass
    return None, 0, 0

# ─── INDICATORS ─────────────────────────────────────────────────────────────────
def calc_ema(series, period):
    return series.ewm(span=period, adjust=False).mean()

def get_first_15min_candle(df_5min):
    """Build first 15-min candle from first 3 x 5-min candles of the day."""
    today = df_5min["timestamp"].dt.date.max()
    today_df = df_5min[df_5min["timestamp"].dt.date == today].head(3)
    if len(today_df) < 3:
        return None
    return {
        "open": today_df.iloc[0]["open"],
        "high": today_df["high"].max(),
        "low": today_df["low"].min(),
        "close": today_df.iloc[-1]["close"],
        "volume": today_df["volume"].sum()
    }

def check_conditions(df, pcr=None):
    """
    Returns dict with breakout/breakdown signals and individual condition flags.
    """
    if df is None or len(df) < 30:
        return None
    
    df = df.copy()
    df["ema9"] = calc_ema(df["close"], 9)
    df["ema21"] = calc_ema(df["close"], 21)
    
    # Average volume (last 20 candles excluding today's)
    avg_volume = df["volume"].iloc[-21:-1].mean() if len(df) > 21 else df["volume"].mean()
    
    last = df.iloc[-1]
    current_close = last["close"]
    current_vol = last["volume"]
    current_ema9 = last["ema9"]
    current_ema21 = last["ema21"]
    current_oi = last.get("oi", 0)
    
    # First 15-min candle
    first15 = get_first_15min_candle(df)
    if not first15:
        return None
    
    # ── CONDITION CHECKS ──
    # Breakout
    bo_c1 = current_close > first15["high"]           # Close above 15-min high
    bo_c2 = current_ema9 > current_ema21              # EMA9 > EMA21
    bo_c3 = current_vol > avg_volume * 1.5            # Volume > 1.5x average
    bo_c4_pcr = (pcr is not None and pcr > 1.0)      # PCR > 1 = bullish OI
    
    # Breakdown (exact opposite except volume — still must be high)
    bd_c1 = current_close < first15["low"]            # Close below 15-min low
    bd_c2 = current_ema9 < current_ema21              # EMA9 < EMA21
    bd_c3 = current_vol > avg_volume * 1.5            # Volume still high (same rule)
    bd_c4_pcr = (pcr is not None and pcr < 1.0)      # PCR < 1 = bearish OI

    is_breakout = bo_c1 and bo_c2 and bo_c3 and bo_c4_pcr
    is_breakdown = bd_c1 and bd_c2 and bd_c3 and bd_c4_pcr

    # Volume ratio
    vol_ratio = current_vol / avg_volume if avg_volume > 0 else 0
    
    return {
        "close": round(current_close, 2),
        "ema9": round(current_ema9, 2),
        "ema21": round(current_ema21, 2),
        "volume": int(current_vol),
        "avg_volume": int(avg_volume),
        "vol_ratio": round(vol_ratio, 2),
        "pcr": round(pcr, 3) if pcr else None,
        "first15_high": round(first15["high"], 2),
        "first15_low": round(first15["low"], 2),
        "breakout": is_breakout,
        "breakdown": is_breakdown,
        # Individual flags
        "bo_c1": bo_c1, "bo_c2": bo_c2, "bo_c3": bo_c3, "bo_c4": bo_c4_pcr,
        "bd_c1": bd_c1, "bd_c2": bd_c2, "bd_c3": bd_c3, "bd_c4": bd_c4_pcr,
    }

# ─── MOCK DATA (for demo without token) ─────────────────────────────────────────
def generate_mock_result(symbol, seed=None):
    """Generate realistic mock scanner result for demo mode."""
    rng = np.random.default_rng(seed or hash(symbol) % 2**31)
    base = rng.uniform(100, 3000)
    close = round(base * rng.uniform(0.97, 1.04), 2)
    ema9 = round(close * rng.uniform(0.98, 1.02), 2)
    ema21 = round(close * rng.uniform(0.97, 1.03), 2)
    first15_h = round(base * 1.01, 2)
    first15_l = round(base * 0.99, 2)
    avg_vol = int(rng.integers(50000, 500000))
    vol = int(avg_vol * rng.uniform(0.5, 3.0))
    pcr = round(rng.uniform(0.5, 2.0), 3)
    vol_ratio = round(vol / avg_vol, 2)
    
    bo_c1 = close > first15_h
    bo_c2 = ema9 > ema21
    bo_c3 = vol_ratio > 1.5
    bo_c4 = pcr > 1.0
    bd_c1 = close < first15_l
    bd_c2 = ema9 < ema21
    bd_c3 = vol_ratio > 1.5
    bd_c4 = pcr < 1.0
    
    return {
        "symbol": symbol,
        "close": close,
        "ema9": ema9,
        "ema21": ema21,
        "volume": vol,
        "avg_volume": avg_vol,
        "vol_ratio": vol_ratio,
        "pcr": pcr,
        "first15_high": first15_h,
        "first15_low": first15_l,
        "breakout": bo_c1 and bo_c2 and bo_c3 and bo_c4,
        "breakdown": bd_c1 and bd_c2 and bd_c3 and bd_c4,
        "bo_c1": bo_c1, "bo_c2": bo_c2, "bo_c3": bo_c3, "bo_c4": bo_c4,
        "bd_c1": bd_c1, "bd_c2": bd_c2, "bd_c3": bd_c3, "bd_c4": bd_c4,
    }

# ─── SESSION STATE ──────────────────────────────────────────────────────────────
if "scan_results" not in st.session_state:
    st.session_state.scan_results = []
if "last_scan_time" not in st.session_state:
    st.session_state.last_scan_time = None
if "is_scanning" not in st.session_state:
    st.session_state.is_scanning = False
if "demo_mode" not in st.session_state:
    st.session_state.demo_mode = True

# ─── SIDEBAR ───────────────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown("""
    <div style='display:flex;align-items:center;gap:10px;padding:12px 0 20px 0;border-bottom:1px solid rgba(255,255,255,0.08);margin-bottom:20px;'>
      <div style='font-size:24px;'>📊</div>
      <div>
        <div style='font-size:15px;font-weight:700;color:#e2e8f0;'>FnO Scanner</div>
        <div style='font-size:11px;color:#64748b;'>Breakout & Breakdown</div>
      </div>
    </div>
    """, unsafe_allow_html=True)

    st.markdown("## ⚙️ Configuration")
    
    demo_mode = st.toggle("Demo Mode (No API Token)", value=st.session_state.demo_mode)
    st.session_state.demo_mode = demo_mode
    
    if not demo_mode:
        access_token = st.text_input("Upstox Access Token", type="password", placeholder="Bearer token...")
    else:
        access_token = "DEMO"
        st.info("Running with simulated data. Toggle off to use live Upstox API.", icon="ℹ️")

    st.markdown("---")
    st.markdown("## 🎛️ Scanner Settings")
    
    scan_interval = st.selectbox("Scan Interval", ["Every 5 min", "Every 1 min", "Manual Only"], index=0)
    vol_multiplier = st.slider("Volume Threshold (x avg)", 1.0, 3.0, 1.5, 0.1,
                                help="Volume must be this many times the 20-candle average")
    pcr_bull_threshold = st.slider("PCR Bull Threshold (>)", 0.5, 2.0, 1.0, 0.1,
                                    help="PCR above this = Bullish OI for breakout")
    
    st.markdown("---")
    st.markdown("## 📋 Market Info")
    now = datetime.now()
    is_weekday = now.weekday() < 5
    is_market_hours = MARKET_OPEN <= now.time() <= MARKET_CLOSE
    is_scanner_active = SCANNER_START <= now.time() <= MARKET_CLOSE
    
    market_status = "🟢 Open" if (is_weekday and is_market_hours) else "🔴 Closed"
    scanner_status = "🟡 Active" if (is_weekday and is_scanner_active) else "⏸️ Waiting"
    
    st.markdown(f"**Market:** {market_status}")
    st.markdown(f"**Scanner:** {scanner_status}")
    st.markdown(f"**Time:** {now.strftime('%H:%M:%S')}")
    st.markdown(f"**Date:** {now.strftime('%a, %d %b %Y')}")

    st.markdown("---")
    if st.button("🔄 Run Scan Now", use_container_width=True):
        st.session_state.run_scan = True
    
    if st.button("🗑️ Clear Results", use_container_width=True):
        st.session_state.scan_results = []
        st.session_state.last_scan_time = None
        st.rerun()

# ─── MAIN CONTENT ───────────────────────────────────────────────────────────────
st.markdown("""
<div style='padding:4px 0 20px 0;'>
  <h1 style='font-size:22px;font-weight:700;color:#e2e8f0;margin:0;'>
    FnO Breakout / Breakdown Scanner
  </h1>
  <p style='color:#64748b;font-size:13px;margin:4px 0 0 0;'>
    Scans NSE FnO stocks every 5 minutes from 9:30 AM · Mon–Fri
  </p>
</div>
""", unsafe_allow_html=True)

# Status bar
now = datetime.now()
is_weekday = now.weekday() < 5
in_scanner_window = SCANNER_START <= now.time() <= MARKET_CLOSE
dot_class = "active" if (is_weekday and in_scanner_window) else ("waiting" if is_weekday else "inactive")
status_text = "Scanner active — running every 5 min" if (is_weekday and in_scanner_window) else \
              ("Market opens at 9:30 AM" if is_weekday else "Market closed — weekday only")

st.markdown(f"""
<div class="status-bar">
  <span class="status-dot {dot_class}"></span>
  <span>{status_text}</span>
  <span style='margin-left:auto;color:#475569;font-size:12px;'>Last scan: {
    st.session_state.last_scan_time.strftime('%H:%M:%S') if st.session_state.last_scan_time else 'Not yet run'
  }</span>
</div>
""", unsafe_allow_html=True)

# ─── SCANNER LOGIC ──────────────────────────────────────────────────────────────
def run_scanner(access_token, demo_mode, vol_mult):
    results = []
    symbols = DEMO_FNO_STOCKS if demo_mode else fetch_fno_instruments(access_token)[:100]
    
    progress = st.progress(0, text="Fetching FnO instruments...")
    
    for i, symbol in enumerate(symbols):
        progress.progress((i + 1) / len(symbols), text=f"Scanning {symbol}...")
        
        if demo_mode:
            result = generate_mock_result(symbol, seed=int(now.timestamp()) // 300 + hash(symbol))
        else:
            instrument_key = f"NSE_EQ|{symbol}"
            df = fetch_candles(access_token, instrument_key)
            underlying_key = f"NSE_EQ|{symbol}"
            pcr, ce_oi, pe_oi = fetch_option_chain_oi(access_token, underlying_key)
            cond = check_conditions(df, pcr)
            if not cond:
                continue
            result = {"symbol": symbol, **cond}
        
        results.append(result)
    
    progress.empty()
    return results

# Trigger scan
if getattr(st.session_state, "run_scan", False):
    st.session_state.run_scan = False
    with st.spinner("Scanning FnO stocks..."):
        results = run_scanner(access_token, demo_mode, vol_multiplier)
        st.session_state.scan_results = results
        st.session_state.last_scan_time = datetime.now()

# Auto-scan if in market hours (manual trigger for Streamlit)
if is_weekday and in_scanner_window and not st.session_state.scan_results:
    if st.button("▶️ Start Scanner", use_container_width=False):
        st.session_state.run_scan = True
        st.rerun()

# ─── DISPLAY RESULTS ────────────────────────────────────────────────────────────
results = st.session_state.scan_results

if results:
    breakouts = [r for r in results if r.get("breakout")]
    breakdowns = [r for r in results if r.get("breakdown")]
    watchlist = [r for r in results if not r.get("breakout") and not r.get("breakdown")]
    
    # Metrics row
    col1, col2, col3, col4 = st.columns(4)
    with col1:
        st.markdown(f"""<div class="metric-card">
          <div class="label">Total Scanned</div>
          <div class="value blue">{len(results)}</div>
        </div>""", unsafe_allow_html=True)
    with col2:
        st.markdown(f"""<div class="metric-card">
          <div class="label">🚀 Breakouts</div>
          <div class="value green">{len(breakouts)}</div>
        </div>""", unsafe_allow_html=True)
    with col3:
        st.markdown(f"""<div class="metric-card">
          <div class="label">📉 Breakdowns</div>
          <div class="value red">{len(breakdowns)}</div>
        </div>""", unsafe_allow_html=True)
    with col4:
        st.markdown(f"""<div class="metric-card">
          <div class="label">⏳ Watching</div>
          <div class="value gold">{len(watchlist)}</div>
        </div>""", unsafe_allow_html=True)
    
    st.markdown("<br>", unsafe_allow_html=True)
    
    # ── BREAKOUTS TABLE ──
    st.markdown(f"""<div class="section-header">
      <span class="icon">🚀</span>
      <span class="title">Breakout Stocks</span>
      <span class="count">{len(breakouts)}</span>
    </div>""", unsafe_allow_html=True)
    
    if breakouts:
        bo_df = pd.DataFrame(breakouts)[["symbol","close","ema9","ema21","first15_high","vol_ratio","pcr","bo_c1","bo_c2","bo_c3","bo_c4"]]
        bo_df.columns = ["Symbol","CMP","EMA 9","EMA 21","15min High","Vol Ratio","PCR","Close>15H","EMA9>21","Vol✓","OI Bull"]
        bo_df[["Close>15H","EMA9>21","Vol✓","OI Bull"]] = bo_df[["Close>15H","EMA9>21","Vol✓","OI Bull"]].applymap(lambda x: "✅" if x else "❌")
        st.dataframe(bo_df, use_container_width=True, hide_index=True)
    else:
        st.markdown('<div style="color:#475569;padding:20px;text-align:center;background:#1a1c23;border-radius:8px;border:1px solid rgba(255,255,255,0.06);">No breakout stocks found in this scan cycle.</div>', unsafe_allow_html=True)
    
    st.markdown("<br>", unsafe_allow_html=True)
    
    # ── BREAKDOWNS TABLE ──
    st.markdown(f"""<div class="section-header">
      <span class="icon">📉</span>
      <span class="title">Breakdown Stocks</span>
      <span class="count">{len(breakdowns)}</span>
    </div>""", unsafe_allow_html=True)
    
    if breakdowns:
        bd_df = pd.DataFrame(breakdowns)[["symbol","close","ema9","ema21","first15_low","vol_ratio","pcr","bd_c1","bd_c2","bd_c3","bd_c4"]]
        bd_df.columns = ["Symbol","CMP","EMA 9","EMA 21","15min Low","Vol Ratio","PCR","Close<15L","EMA9<21","Vol✓","OI Bear"]
        bd_df[["Close<15L","EMA9<21","Vol✓","OI Bear"]] = bd_df[["Close<15L","EMA9<21","Vol✓","OI Bear"]].applymap(lambda x: "✅" if x else "❌")
        st.dataframe(bd_df, use_container_width=True, hide_index=True)
    else:
        st.markdown('<div style="color:#475569;padding:20px;text-align:center;background:#1a1c23;border-radius:8px;border:1px solid rgba(255,255,255,0.06);">No breakdown stocks found in this scan cycle.</div>', unsafe_allow_html=True)
    
    st.markdown("<br>", unsafe_allow_html=True)
    
    # ── WATCHLIST / NEAR SIGNALS ──
    near = [r for r in watchlist if 
            (abs(r["close"] - r["first15_high"]) / r["first15_high"] < 0.005) or
            (abs(r["close"] - r["first15_low"]) / r["first15_low"] < 0.005)]
    
    if near:
        st.markdown(f"""<div class="section-header">
          <span class="icon">👁️</span>
          <span class="title">Near Signal (within 0.5% of 15-min boundary)</span>
          <span class="count">{len(near)}</span>
        </div>""", unsafe_allow_html=True)
        near_df = pd.DataFrame(near)[["symbol","close","first15_high","first15_low","vol_ratio","pcr"]]
        near_df.columns = ["Symbol","CMP","15min High","15min Low","Vol Ratio","PCR"]
        st.dataframe(near_df, use_container_width=True, hide_index=True)
    
    # ── CONDITION LOGIC EXPLANATION ──
    st.markdown("<br>", unsafe_allow_html=True)
    with st.expander("📘 Scanner Condition Logic"):
        st.markdown("""
**Breakout — All 4 conditions must be TRUE:**
| # | Condition | Logic |
|---|---|---|
| 1 | **5-min close > 15-min high** | Current 5-min candle close > high of first 15-min candle (9:15–9:30) |
| 2 | **EMA 9 > EMA 21** | Bullish EMA crossover on 5-min chart |
| 3 | **Volume > 1.5x avg** | Current candle volume > 1.5× 20-candle average |
| 4 | **PCR > 1.0 (Bullish OI)** | Put-Call Ratio of ATM options > 1 = more PE writing = bullish bias |

**Breakdown — All 4 conditions must be TRUE:**
| # | Condition | Logic |
|---|---|---|
| 1 | **5-min close < 15-min low** | Current 5-min candle close < low of first 15-min candle |
| 2 | **EMA 9 < EMA 21** | Bearish EMA crossover on 5-min chart |
| 3 | **Volume > 1.5x avg** | Same volume condition (selling pressure must be strong) |
| 4 | **PCR < 1.0 (Bearish OI)** | PCR below 1 = more CE writing = bearish bias |

**OI Signal Logic:** PCR (Put-Call Ratio) = Total PE OI ÷ Total CE OI for nearest expiry ATM options.
- PCR > 1 → PE writers are dominant → Market makers expect upside → **Bullish**
- PCR < 1 → CE writers are dominant → Market makers expect downside → **Bearish**
        """)

else:
    # Empty state
    st.markdown("""
    <div style='text-align:center;padding:60px 20px;'>
      <div style='font-size:48px;margin-bottom:16px;'>📡</div>
      <div style='font-size:18px;font-weight:600;color:#e2e8f0;margin-bottom:8px;'>Scanner Ready</div>
      <div style='color:#64748b;font-size:14px;max-width:400px;margin:0 auto;'>
        Click "Run Scan Now" in the sidebar to start scanning FnO stocks for breakout and breakdown signals.
      </div>
    </div>
    """, unsafe_allow_html=True)

# Auto-refresh note
if is_weekday and in_scanner_window:
    st.markdown("---")
    st.markdown(f'<div style="color:#475569;font-size:12px;text-align:center;">Auto-refresh: Click "Run Scan Now" or press R to rescan · {now.strftime("%d %b %Y, %H:%M:%S")}</div>', unsafe_allow_html=True)
