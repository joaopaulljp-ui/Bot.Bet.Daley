“””
╔══════════════════════════════════════════════════════════╗
║     BETFAIR DELAY BOT — Versão Railway / iPhone         ║
║     Interface web integrada — acesse pelo Safari        ║
╚══════════════════════════════════════════════════════════╝
“””

import os
import json
import time
import threading
import datetime
import requests
from flask import Flask, jsonify, request, render_template_string

app = Flask(**name**)

# ─────────────────────────────────────────

# CONFIGURAÇÃO VIA VARIÁVEIS DE AMBIENTE

# (definidas no Railway — nunca no código)

# ─────────────────────────────────────────

CONFIG = {
“app_key”:                  os.environ.get(“BETFAIR_APP_KEY”, “”),
“username”:                 os.environ.get(“BETFAIR_USERNAME”, “”),
“password”:                 os.environ.get(“BETFAIR_PASSWORD”, “”),
“stake”:              float(os.environ.get(“STAKE”, “50”)),
“min_odd”:            float(os.environ.get(“MIN_ODD”, “1.5”)),
“max_odd”:            float(os.environ.get(“MAX_ODD”, “4.0”)),
“delay_threshold”:    float(os.environ.get(“DELAY_THRESHOLD”, “4”)),
“check_interval”:     float(os.environ.get(“CHECK_INTERVAL”, “1.5”)),
“dry_run”:                  os.environ.get(“DRY_RUN”, “true”).lower() == “true”,
}

BETFAIR_LOGIN_URL = “https://identitysso-cert.betfair.com/api/login”
BETTING_API       = “https://api.betfair.com/exchange/betting/json-rpc/v1”
ACCOUNT_API       = “https://api.betfair.com/exchange/account/json-rpc/v1”

# ─────────────────────────────────────────

# ESTADO GLOBAL DO BOT

# ─────────────────────────────────────────

state = {
“running”:        False,
“session_token”:  None,
“headers”:        {},
“balance”:        0.0,
“markets”:        [],
“monitored”:      {},
“bets”:           [],
“log”:            [],
“total_stake”:    0.0,
“total_profit”:   0.0,
“opportunities”:  0,
}

# ══════════════════════════════════════════

# FUNÇÕES DO BOT

# ══════════════════════════════════════════

def add_log(msg, level=“INFO”):
entry = {
“time”: datetime.datetime.now().strftime(”%H:%M:%S”),
“level”: level,
“msg”: msg
}
state[“log”].insert(0, entry)
if len(state[“log”]) > 100:
state[“log”].pop()
print(f”[{entry[‘time’]}] [{level}] {msg}”)

def login():
add_log(“Autenticando na Betfair…”)
if not CONFIG[“app_key”] or not CONFIG[“username”]:
add_log(“❌ Credenciais não configuradas. Veja as variáveis de ambiente no Railway.”, “ERROR”)
return False
try:
resp = requests.post(
BETFAIR_LOGIN_URL,
data={“username”: CONFIG[“username”], “password”: CONFIG[“password”]},
headers={“X-Application”: CONFIG[“app_key”], “Content-Type”: “application/x-www-form-urlencoded”},
timeout=10
)
data = resp.json()
if data.get(“status”) == “SUCCESS”:
state[“session_token”] = data[“token”]
state[“headers”] = {
“X-Application”:  CONFIG[“app_key”],
“X-Authentication”: state[“session_token”],
“Content-Type”:   “application/json”
}
add_log(“✅ Login realizado com sucesso!”)
return True
else:
add_log(f”❌ Falha no login: {data.get(‘error’, ‘erro desconhecido’)}”, “ERROR”)
return False
except Exception as e:
add_log(f”❌ Erro de conexão: {e}”, “ERROR”)
return False

def api_call(method, params, base=BETTING_API):
payload = json.dumps([{
“jsonrpc”: “2.0”,
“method”:  f”SportsAPING/v1.0/{method}”,
“params”:  params,
“id”: 1
}])
try:
resp = requests.post(base, data=payload, headers=state[“headers”], timeout=10)
result = resp.json()
if isinstance(result, list) and “result” in result[0]:
return result[0][“result”]
except Exception as e:
add_log(f”Erro API {method}: {e}”, “ERROR”)
return None

def get_balance():
result = api_call(“getAccountFunds”, {“filter”: {}},
base=“https://api.betfair.com/exchange/account/json-rpc/v1”)
if result:
state[“balance”] = result.get(“availableToBetBalance”, 0)

def get_live_markets():
result = api_call(“listMarketCatalogue”, {
“filter”: {
“eventTypeIds”: [“1”],
“inPlayOnly”: True,
“marketTypeCodes”: [“MATCH_ODDS”]
},
“marketProjection”: [“MARKET_NAME”, “EVENT”, “RUNNER_DESCRIPTION”],
“maxResults”: 20,
“sort”: “FIRST_TO_START”
})
if result:
state[“markets”] = result
add_log(f”⚽ {len(result)} jogo(s) ao vivo encontrado(s)”)
return result or []

def check_market(market):
mid  = market[“marketId”]
name = market.get(“event”, {}).get(“name”, mid)
now  = time.time()

```
result = api_call("listMarketBook", {
    "marketIds": [mid],
    "priceProjection": {
        "priceData": ["EX_BEST_OFFERS"],
        "exBestOffersOverrides": {"bestPricesDepth": 3}
    }
})
if not result:
    return

book = result[0]
if book.get("status") != "OPEN":
    return

for runner in book.get("runners", []):
    if runner.get("status") != "ACTIVE":
        continue
    backs = runner.get("ex", {}).get("availableToBack", [])
    if not backs:
        continue

    odd       = backs[0]["price"]
    available = backs[0]["size"]
    rid       = runner["selectionId"]

    if not (CONFIG["min_odd"] <= odd <= CONFIG["max_odd"]):
        continue

    key = f"{mid}_{rid}"
    if key not in state["monitored"]:
        state["monitored"][key] = {
            "last_odd": odd, "stable_since": now, "history": [odd]
        }
        continue

    rec = state["monitored"][key]
    if abs(odd - rec["last_odd"]) > 0.01:
        rec["last_odd"]     = odd
        rec["stable_since"] = now
        rec["history"].append(odd)
        if len(rec["history"]) > 20:
            rec["history"].pop(0)
        continue

    stable = now - rec["stable_since"]
    if stable < CONFIG["delay_threshold"]:
        continue
    if available < 20:
        continue

    hist = rec["history"]
    vol  = 0
    if len(hist) >= 3:
        avg = sum(hist) / len(hist)
        vol = (sum((x - avg)**2 for x in hist) / len(hist)) ** 0.5

    conf = min(int(stable / 10 * 40), 40)
    conf += min(int(vol / 0.5 * 30), 30)
    conf += min(int(available / 500 * 30), 30)

    state["opportunities"] += 1
    runner_name = _runner_name(market, rid)
    add_log(
        f"⚡ DELAY | {name} | {runner_name} @ {odd:.2f} | "
        f"Estável {stable:.1f}s | Confiança {conf}/100",
        "OPPORTUNITY"
    )

    if conf >= 60:
        place_bet(mid, rid, odd, CONFIG["stake"], name, runner_name)
```

def _runner_name(market, runner_id):
for r in market.get(“runners”, []):
if r.get(“selectionId”) == runner_id:
return r.get(“runnerName”, str(runner_id))
return str(runner_id)

def place_bet(mid, rid, odd, stake, market_name, runner_name):
mode = “DRY RUN” if CONFIG[“dry_run”] else “REAL”
bet = {
“time”:    datetime.datetime.now().strftime(”%H:%M:%S”),
“market”:  market_name,
“runner”:  runner_name,
“odd”:     odd,
“stake”:   stake,
“return”:  round(stake * odd, 2),
“mode”:    mode,
“status”:  “pending”
}

```
if not CONFIG["dry_run"]:
    result = api_call("placeOrders", {
        "marketId": mid,
        "instructions": [{
            "selectionId": rid,
            "handicap": 0,
            "side": "BACK",
            "orderType": "LIMIT",
            "limitOrder": {
                "size": stake,
                "price": odd,
                "persistenceType": "LAPSE"
            }
        }]
    })
    if result:
        bet["status"] = "placed"
        state["total_stake"]  += stake
        add_log(f"✅ APOSTA REAL | R${stake:.2f} @ {odd:.2f} | Retorno: R${stake*odd:.2f}", "BET")
    else:
        bet["status"] = "failed"
        add_log("❌ Falha ao colocar aposta", "ERROR")
else:
    bet["status"] = "simulated"
    add_log(f"🧪 SIMULADO | R${stake:.2f} @ {odd:.2f} | Retorno: R${stake*odd:.2f}", "BET")

state["bets"].insert(0, bet)
if len(state["bets"]) > 200:
    state["bets"].pop()
```

# ══════════════════════════════════════════

# LOOP DO BOT (roda em thread separada)

# ══════════════════════════════════════════

def bot_loop():
if not login():
state[“running”] = False
return

```
get_balance()
markets = []
cycle   = 0

while state["running"]:
    try:
        cycle += 1
        if cycle % 20 == 1:
            markets = get_live_markets()
            get_balance()

        for m in markets:
            check_market(m)

        time.sleep(CONFIG["check_interval"])
    except Exception as e:
        add_log(f"Erro no loop: {e}", "ERROR")
        time.sleep(5)

add_log("🛑 Bot encerrado.")
```

# ══════════════════════════════════════════

# INTERFACE WEB (acessada pelo iPhone)

# ══════════════════════════════════════════

HTML = “””<!DOCTYPE html>

<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<title>Delay Bot</title>
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;700&family=Manrope:wght@400;600;800&display=swap" rel="stylesheet">
<style>
  :root {
    --bg: #07090f;
    --card: #0e1219;
    --border: #1a2333;
    --green: #00e5a0;
    --red: #ff4757;
    --yellow: #ffd740;
    --blue: #4fc3f7;
    --text: #dde6f0;
    --muted: #4a6070;
    --font: 'Manrope', sans-serif;
    --mono: 'IBM Plex Mono', monospace;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; -webkit-tap-highlight-color: transparent; }
  body { background: var(--bg); color: var(--text); font-family: var(--font); min-height: 100vh; padding-bottom: 80px; }

/* TOP BAR */
.topbar {
position: sticky; top: 0; z-index: 100;
background: rgba(7,9,15,0.95);
backdrop-filter: blur(12px);
-webkit-backdrop-filter: blur(12px);
border-bottom: 1px solid var(–border);
padding: 14px 16px;
display: flex; align-items: center; justify-content: space-between;
}
.topbar-title { font-size: 17px; font-weight: 800; letter-spacing: -0.3px; }
.topbar-title span { color: var(–green); }
.live-dot {
display: flex; align-items: center; gap: 6px;
font-size: 11px; font-family: var(–mono); color: var(–green);
}
.dot { width: 7px; height: 7px; border-radius: 50%; background: var(–green); animation: blink 1.4s infinite; }
@keyframes blink { 0%,100%{opacity:1} 50%{opacity:0.3} }

/* CONTENT */
.content { padding: 16px; display: flex; flex-direction: column; gap: 14px; }

/* CARDS */
.card { background: var(–card); border: 1px solid var(–border); border-radius: 14px; overflow: hidden; }
.card-header {
padding: 12px 16px;
border-bottom: 1px solid var(–border);
font-size: 11px; font-family: var(–mono);
color: var(–muted); letter-spacing: 1.5px; text-transform: uppercase;
display: flex; align-items: center; justify-content: space-between;
}
.card-body { padding: 16px; }

/* STATS ROW */
.stats { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; }
.stat { background: var(–card); border: 1px solid var(–border); border-radius: 12px; padding: 14px; }
.stat-val { font-size: 26px; font-weight: 800; font-family: var(–mono); color: var(–green); }
.stat-val.red { color: var(–red); }
.stat-val.yellow { color: var(–yellow); }
.stat-lbl { font-size: 11px; color: var(–muted); margin-top: 3px; }

/* TOGGLE */
.toggle-row { display: flex; align-items: center; justify-content: space-between; padding: 16px; }
.toggle-info h3 { font-size: 15px; font-weight: 700; }
.toggle-info p { font-size: 12px; color: var(–muted); margin-top: 2px; }
.toggle {
position: relative; width: 52px; height: 30px;
background: var(–border); border-radius: 15px; cursor: pointer;
transition: background 0.3s;
}
.toggle.on { background: var(–green); }
.toggle::after {
content: ‘’; position: absolute;
width: 24px; height: 24px; border-radius: 50%;
background: #fff; top: 3px; left: 3px;
transition: transform 0.3s;
box-shadow: 0 2px 6px rgba(0,0,0,0.4);
}
.toggle.on::after { transform: translateX(22px); }

/* MODE BADGE */
.mode-badge {
display: inline-flex; align-items: center; gap: 6px;
padding: 5px 12px; border-radius: 20px; font-size: 12px; font-weight: 700;
}
.mode-badge.sim { background: rgba(255,215,64,0.12); color: var(–yellow); border: 1px solid rgba(255,215,64,0.3); }
.mode-badge.real { background: rgba(255,71,87,0.12); color: var(–red); border: 1px solid rgba(255,71,87,0.3); }

/* LOG */
.log-list { display: flex; flex-direction: column; gap: 6px; max-height: 320px; overflow-y: auto; }
.log-item {
padding: 10px 12px; border-radius: 8px; border: 1px solid var(–border);
font-size: 12px; line-height: 1.5;
display: flex; gap: 10px; align-items: flex-start;
}
.log-item.OPPORTUNITY { border-color: rgba(0,229,160,0.3); background: rgba(0,229,160,0.05); }
.log-item.BET { border-color: rgba(79,195,247,0.3); background: rgba(79,195,247,0.05); }
.log-item.ERROR { border-color: rgba(255,71,87,0.3); background: rgba(255,71,87,0.05); }
.log-time { font-family: var(–mono); font-size: 10px; color: var(–muted); white-space: nowrap; flex-shrink: 0; }

/* BETS TABLE */
.bet-list { display: flex; flex-direction: column; gap: 8px; }
.bet-item { padding: 12px; border-radius: 10px; border: 1px solid var(–border); }
.bet-top { display: flex; justify-content: space-between; align-items: center; margin-bottom: 6px; }
.bet-market { font-size: 13px; font-weight: 700; }
.bet-runner { font-size: 12px; color: var(–muted); margin-bottom: 6px; }
.bet-bottom { display: flex; justify-content: space-between; font-size: 12px; font-family: var(–mono); }
.bet-odd { color: var(–green); font-weight: 700; }
.bet-return { color: var(–yellow); }
.badge { display: inline-block; padding: 2px 8px; border-radius: 4px; font-size: 10px; font-weight: 700; font-family: var(–mono); }
.badge.sim { background: rgba(255,215,64,0.15); color: var(–yellow); }
.badge.placed { background: rgba(0,229,160,0.15); color: var(–green); }
.badge.failed { background: rgba(255,71,87,0.15); color: var(–red); }

/* MARKETS */
.market-item { padding: 10px 12px; border-bottom: 1px solid var(–border); font-size: 13px; display: flex; align-items: center; gap: 8px; }
.market-item:last-child { border-bottom: none; }
.market-dot { width: 6px; height: 6px; border-radius: 50%; background: var(–green); flex-shrink: 0; }

/* BOTTOM NAV */
.bottom-nav {
position: fixed; bottom: 0; left: 0; right: 0;
background: rgba(14,18,25,0.97);
backdrop-filter: blur(16px);
-webkit-backdrop-filter: blur(16px);
border-top: 1px solid var(–border);
display: flex;
padding-bottom: env(safe-area-inset-bottom);
}
.nav-btn {
flex: 1; padding: 12px 8px; text-align: center;
cursor: pointer; color: var(–muted); font-size: 10px;
transition: color 0.2s;
}
.nav-btn.active { color: var(–green); }
.nav-btn .nav-icon { font-size: 20px; display: block; margin-bottom: 2px; }

/* TABS */
.tab-content { display: none; }
.tab-content.active { display: flex; flex-direction: column; gap: 14px; }

/* CONFIG */
.config-item { padding: 14px 16px; border-bottom: 1px solid var(–border); display: flex; justify-content: space-between; align-items: center; }
.config-item:last-child { border-bottom: none; }
.config-label { font-size: 13px; font-weight: 600; }
.config-sub { font-size: 11px; color: var(–muted); margin-top: 2px; }
.config-val { font-family: var(–mono); font-size: 14px; color: var(–green); font-weight: 700; }

.empty { text-align: center; padding: 32px 16px; color: var(–muted); font-size: 13px; }
</style>

</head>
<body>

<div class="topbar">
  <div class="topbar-title">Delay<span>Bot</span></div>
  <div class="live-dot" id="live-indicator">
    <div class="dot"></div> OFFLINE
  </div>
</div>

<div class="content">

  <!-- TAB: DASHBOARD -->

  <div id="tab-dashboard" class="tab-content active">

```
<!-- STATS -->
<div class="stats">
  <div class="stat">
    <div class="stat-val" id="s-balance">R$0</div>
    <div class="stat-lbl">Saldo Betfair</div>
  </div>
  <div class="stat">
    <div class="stat-val yellow" id="s-opps">0</div>
    <div class="stat-lbl">Oportunidades</div>
  </div>
  <div class="stat">
    <div class="stat-val" id="s-bets">0</div>
    <div class="stat-lbl">Apostas</div>
  </div>
  <div class="stat">
    <div class="stat-val" id="s-markets">0</div>
    <div class="stat-lbl">Jogos ao vivo</div>
  </div>
</div>

<!-- BOT CONTROL -->
<div class="card">
  <div class="card-header">
    Controle do Bot
    <span id="mode-badge" class="mode-badge sim">🧪 SIMULAÇÃO</span>
  </div>
  <div class="toggle-row">
    <div class="toggle-info">
      <h3 id="bot-status-text">Bot parado</h3>
      <p id="bot-status-sub">Toque para iniciar o monitoramento</p>
    </div>
    <div class="toggle" id="bot-toggle" onclick="toggleBot()"></div>
  </div>
</div>

<!-- MERCADOS AO VIVO -->
<div class="card">
  <div class="card-header">⚽ Jogos Monitorados</div>
  <div id="markets-list">
    <div class="empty">Inicie o bot para ver os jogos ao vivo</div>
  </div>
</div>

<!-- LOG -->
<div class="card">
  <div class="card-header">📋 Log em tempo real</div>
  <div class="card-body" style="padding:10px">
    <div class="log-list" id="log-list">
      <div class="empty">Aguardando início...</div>
    </div>
  </div>
</div>
```

  </div>

  <!-- TAB: APOSTAS -->

  <div id="tab-bets" class="tab-content">
    <div class="card">
      <div class="card-header">💰 Histórico de Apostas</div>
      <div class="card-body" style="padding:10px">
        <div class="bet-list" id="bet-list">
          <div class="empty">Nenhuma aposta ainda</div>
        </div>
      </div>
    </div>
  </div>

  <!-- TAB: CONFIG -->

  <div id="tab-config" class="tab-content">
    <div class="card">
      <div class="card-header">⚙️ Configurações Ativas</div>
      <div id="config-list"></div>
    </div>
    <div class="card">
      <div class="card-header">ℹ️ Como alterar</div>
      <div class="card-body">
        <p style="font-size:13px;color:var(--muted);line-height:1.7">
          Para alterar qualquer configuração, acesse o painel do <strong style="color:var(--text)">Railway</strong>, vá em <strong style="color:var(--text)">Variables</strong> e edite o valor desejado. O bot reinicia automaticamente.
        </p>
      </div>
    </div>
  </div>

</div>

<!-- BOTTOM NAV -->

<div class="bottom-nav">
  <div class="nav-btn active" id="nav-dashboard" onclick="switchTab('dashboard')">
    <span class="nav-icon">📊</span>Dashboard
  </div>
  <div class="nav-btn" id="nav-bets" onclick="switchTab('bets')">
    <span class="nav-icon">💰</span>Apostas
  </div>
  <div class="nav-btn" id="nav-config" onclick="switchTab('config')">
    <span class="nav-icon">⚙️</span>Config
  </div>
</div>

<script>
let botRunning = false;

function switchTab(name) {
  document.querySelectorAll('.tab-content').forEach(t => t.classList.remove('active'));
  document.querySelectorAll('.nav-btn').forEach(b => b.classList.remove('active'));
  document.getElementById('tab-' + name).classList.add('active');
  document.getElementById('nav-' + name).classList.add('active');
}

function toggleBot() {
  const toggle = document.getElementById('bot-toggle');
  if (!botRunning) {
    fetch('/api/start', {method:'POST'})
      .then(r => r.json())
      .then(d => {
        if (d.ok) {
          botRunning = true;
          toggle.classList.add('on');
          document.getElementById('bot-status-text').textContent = 'Bot rodando';
          document.getElementById('bot-status-sub').textContent = 'Monitorando futebol ao vivo...';
        } else {
          alert('Erro: ' + d.error);
        }
      });
  } else {
    fetch('/api/stop', {method:'POST'})
      .then(r => r.json())
      .then(() => {
        botRunning = false;
        toggle.classList.remove('on');
        document.getElementById('bot-status-text').textContent = 'Bot parado';
        document.getElementById('bot-status-sub').textContent = 'Toque para iniciar o monitoramento';
      });
  }
}

function refresh() {
  fetch('/api/status')
    .then(r => r.json())
    .then(d => {
      // Indicador
      const ind = document.getElementById('live-indicator');
      ind.innerHTML = d.running
        ? '<div class="dot"></div> AO VIVO'
        : '<div class="dot" style="background:var(--muted)"></div> OFFLINE';
      ind.style.color = d.running ? 'var(--green)' : 'var(--muted)';

      // Stats
      document.getElementById('s-balance').textContent  = 'R$' + d.balance.toFixed(0);
      document.getElementById('s-opps').textContent     = d.opportunities;
      document.getElementById('s-bets').textContent     = d.bets_count;
      document.getElementById('s-markets').textContent  = d.markets_count;

      // Mode badge
      const mb = document.getElementById('mode-badge');
      mb.className = 'mode-badge ' + (d.dry_run ? 'sim' : 'real');
      mb.textContent = d.dry_run ? '🧪 SIMULAÇÃO' : '🔴 REAL';

      // Toggle sync
      const toggle = document.getElementById('bot-toggle');
      botRunning = d.running;
      toggle.classList.toggle('on', d.running);

      // Mercados
      const ml = document.getElementById('markets-list');
      if (d.markets.length === 0) {
        ml.innerHTML = '<div class="empty">Nenhum jogo ao vivo no momento</div>';
      } else {
        ml.innerHTML = d.markets.map(m =>
          `<div class="market-item"><div class="market-dot"></div>${m.event?.name || m.marketId}</div>`
        ).join('');
      }

      // Log
      const ll = document.getElementById('log-list');
      if (d.log.length === 0) {
        ll.innerHTML = '<div class="empty">Aguardando início...</div>';
      } else {
        ll.innerHTML = d.log.slice(0,30).map(l =>
          `<div class="log-item ${l.level}">
            <span class="log-time">${l.time}</span>
            <span>${l.msg}</span>
          </div>`
        ).join('');
      }

      // Apostas
      const bl = document.getElementById('bet-list');
      if (d.bets.length === 0) {
        bl.innerHTML = '<div class="empty">Nenhuma aposta ainda</div>';
      } else {
        bl.innerHTML = d.bets.map(b =>
          `<div class="bet-item">
            <div class="bet-top">
              <span class="bet-market">${b.market}</span>
              <span class="badge ${b.status}">${b.status.toUpperCase()}</span>
            </div>
            <div class="bet-runner">${b.runner}</div>
            <div class="bet-bottom">
              <span class="bet-odd">@ ${b.odd.toFixed(2)}</span>
              <span>R$${b.stake.toFixed(2)}</span>
              <span class="bet-return">→ R$${b.return.toFixed(2)}</span>
            </div>
          </div>`
        ).join('');
      }

      // Config
      const cl = document.getElementById('config-list');
      cl.innerHTML = [
        ['Stake por aposta', 'R$' + d.config.stake.toFixed(2)],
        ['Odd mínima', d.config.min_odd.toFixed(2)],
        ['Odd máxima', d.config.max_odd.toFixed(2)],
        ['Threshold delay', d.config.delay_threshold + 's'],
        ['Modo', d.dry_run ? '🧪 Simulação' : '🔴 Real'],
      ].map(([label, val]) =>
        `<div class="config-item">
          <div><div class="config-label">${label}</div></div>
          <div class="config-val">${val}</div>
        </div>`
      ).join('');
    })
    .catch(() => {});
}

setInterval(refresh, 2000);
refresh();
</script>

</body>
</html>
"""

# ══════════════════════════════════════════

# ROTAS DA API

# ══════════════════════════════════════════

@app.route(”/”)
def index():
return render_template_string(HTML)

@app.route(”/api/start”, methods=[“POST”])
def start():
if state[“running”]:
return jsonify({“ok”: True, “msg”: “Já rodando”})
if not CONFIG[“app_key”] or CONFIG[“app_key”] == “”:
return jsonify({“ok”: False, “error”: “Configure BETFAIR_APP_KEY nas variáveis do Railway.”})
state[“running”] = True
t = threading.Thread(target=bot_loop, daemon=True)
t.start()
return jsonify({“ok”: True})

@app.route(”/api/stop”, methods=[“POST”])
def stop():
state[“running”] = False
return jsonify({“ok”: True})

@app.route(”/api/status”)
def status():
return jsonify({
“running”:       state[“running”],
“balance”:       state[“balance”],
“markets”:       state[“markets”],
“markets_count”: len(state[“markets”]),
“log”:           state[“log”][:30],
“bets”:          state[“bets”][:50],
“bets_count”:    len(state[“bets”]),
“opportunities”: state[“opportunities”],
“dry_run”:       CONFIG[“dry_run”],
“config”: {
“stake”:           CONFIG[“stake”],
“min_odd”:         CONFIG[“min_odd”],
“max_odd”:         CONFIG[“max_odd”],
“delay_threshold”: CONFIG[“delay_threshold”],
}
})

# ══════════════════════════════════════════

# INICIAR SERVIDOR

# ══════════════════════════════════════════

if **name** == “**main**”:
port = int(os.environ.get(“PORT”, 5000))
add_log(“🚀 Servidor iniciado — acesse pelo Safari do iPhone”)
app.run(host=“0.0.0.0”, port=port, debug=False)
