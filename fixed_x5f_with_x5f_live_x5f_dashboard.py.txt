
import os
import time
import threading
import urllib.parse
import requests
import json
import hashlib
import random
from flask import Flask, jsonify, render_template_string
from instagrapi import Client
from instagrapi.exceptions import LoginRequired, ChallengeRequired

# --------- CONFIG ----------
SESSION_ID_1 = os.getenv("SESSION_ID_1")
SESSION_ID_2 = os.getenv("SESSION_ID_2")
SESSION_ID_3 = os.getenv("SESSION_ID_3")
SESSION_ID_4 = os.getenv("SESSION_ID_4")
SESSION_ID_5 = os.getenv("SESSION_ID_5")
SESSION_ID_6 = os.getenv("SESSION_ID_6")
GROUP_IDS = os.getenv("GROUP_IDS", "")
MESSAGE_TEXT = os.getenv("MESSAGE_TEXT", "Hello")
SELF_URL = os.getenv("SELF_URL", "")
NC_TITLES_RAW = os.getenv("NC_TITLES", "") 
SPAM_START_OFFSET = int(os.getenv("SPAM_START_OFFSET", "1"))
SPAM_GAP_BETWEEN_ACCOUNTS = int(os.getenv("SPAM_GAP_BETWEEN_ACCOUNTS", "6"))
NC_START_OFFSET = int(os.getenv("NC_START_OFFSET", "1"))
NC_ACC_GAP = int(os.getenv("NC_ACC_GAP", "30"))
MSG_REFRESH_DELAY = int(os.getenv("MSG_REFRESH_DELAY", "1"))
BURST_COUNT = int(os.getenv("BURST_COUNT", "1"))
SELF_PING_INTERVAL = int(os.getenv("SELF_PING_INTERVAL", "60"))
COOLDOWN_ON_ERROR = int(os.getenv("COOLDOWN_ON_ERROR", "300"))
DOC_ID = os.getenv("DOC_ID", "29088580780787855")
CSRF_TOKEN = os.getenv("CSRF_TOKEN", "")
SESSION_KEEPALIVE_INTERVAL = int(os.getenv("SESSION_KEEPALIVE_INTERVAL", "14400"))

STABLE_USER_AGENTS = [
    "Instagram 269.0.0.18.75 Android (33/13; 480dpi; 1080x2400; samsung; SM-G998B; p3s; exynos2100; en_US; 314665256)",
    "Instagram 268.0.0.19.83 Android (32/12; 420dpi; 1080x2280; OnePlus; CPH2449; OP5958L1; qcom; en_US; 314665256)",
    "Instagram 267.0.0.21.93 Android (33/13; 480dpi; 1080x2400; Xiaomi; 2211133C; ruby; mt6895; en_US; 314665256)",
    "Instagram 269.0.0.18.75 Android (31/12; 420dpi; 1080x2400; google; Pixel 6; oriole; gs101; en_US; 314665256)",
    "Instagram 266.0.0.16.75 Android (30/11; 420dpi; 1080x2340; samsung; SM-A525F; a52q; qcom; en_US; 314665256)",
    "Instagram 269.0.0.18.75 Android (33/13; 440dpi; 1080x2400; realme; RMX3710; RE54ABL1; qcom; en_US; 314665256)",
]
PROXIES = {
    "acc1": os.getenv("PROXY_1", ""),
    "acc2": os.getenv("PROXY_2", ""),
    "acc3": os.getenv("PROXY_3", ""),
    "acc4": os.getenv("PROXY_4", ""),
    "acc5": os.getenv("PROXY_5", ""),
    "acc6": os.getenv("PROXY_6", ""),
}

app = Flask(__name__)
MAX_SESSION_LOGS = 300
session_logs = {"acc1": [], "acc2": [], "acc3": [], "acc4": [], "acc5": [], "acc6": [], "system": []}
logs_lock = threading.Lock()
START_TIME = time.time()

def _push_log(session, msg):
    if session not in session_logs: session = "system"
    with logs_lock:
        timestamp = time.strftime('%Y-%m-%d %H:%M:%S')
        session_logs[session].append(f"[{timestamp}] {msg}")
        if len(session_logs[session]) > MAX_SESSION_LOGS:
            session_logs[session].pop(0)

def log(msg, session="system"):
    print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] {msg}", flush=True)
    _push_log(session, msg)

# ========== DASHBOARD HTML ==========
DASHBOARD_HTML = '''
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Insta Bot Live Dashboard</title>
<style>
*{margin:0;padding:0;box-sizing:border-box}
body{background:#0a0a0f;color:#e0e0e0;font-family:'JetBrains Mono',monospace;padding:12px}
.header{background:linear-gradient(135deg,#667eea,#764ba2);padding:16px;border-radius:12px;display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}
.header h1{font-size:18px}
.live-dot{width:10px;height:10px;background:#00ff88;border-radius:50%;display:inline-block;animation:blink 1s infinite;box-shadow:0 0 10px #00ff88}
@keyframes blink{0%,100%{opacity:1}50%{opacity:0.3}}
.stats{display:grid;grid-template-columns:repeat(auto-fit,minmax(140px,1fr));gap:10px;margin-bottom:12px}
.stat-card{background:#16161f;border:1px solid #2a2a3a;border-radius:10px;padding:12px;text-align:center}
.stat-card .val{font-size:22px;font-weight:bold;color:#667eea}
.stat-card .lbl{font-size:11px;color:#888;margin-top:4px}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(300px,1fr));gap:12px}
.card{background:#16161f;border:1px solid #2a2a3a;border-radius:12px;overflow:hidden}
.card-head{padding:10px 14px;background:#1e1e2f;display:flex;justify-content:space-between;align-items:center;font-weight:bold;font-size:13px}
.card-head .badge{padding:2px 8px;border-radius:12px;font-size:10px}
.badge-ok{background:#00ff8822;color:#00ff88;border:1px solid #00ff88}
.badge-warn{background:#ffaa0022;color:#ffaa00;border:1px solid #ffaa00}
.badge-err{background:#ff444422;color:#ff4444;border:1px solid #ff4444}
.logs{height:280px;overflow-y:auto;padding:10px;font-size:11px;line-height:1.6;background:#0f0f17}
.logs div{border-bottom:1px solid #1a1a2a;padding:2px 0}
.log-ok{color:#00ff88}
.log-err{color:#ff5555}
.log-warn{color:#ffaa00}
.log-info{color:#8a8aff}
.log-ping{color:#00d4ff;font-weight:bold}
.system-card{grid-column:1/-1}
.time-bar{background:#16161f;border:1px solid #2a2a3a;border-radius:10px;padding:10px 14px;display:flex;justify-content:space-between;margin-bottom:12px;font-size:12px}
</style>
</head>
<body>
<div class="header">
<h1>🚀 INSTA BOT <span class="live-dot"></span> LIVE</h1>
<div id="clock">--:--:--</div>
</div>

<div class="time-bar">
<div>⏱ Uptime: <b id="uptime">0m</b></div>
<div>🔁 Self-Ping Interval: <b id="pingInterval">--</b>s</div>
<div>📡 Last Ping: <b id="lastPing" style="color:#00d4ff">Waiting...</b></div>
<div>💓 KeepAlive: <b id="keepAlive">--</b>s</div>
</div>

<div class="stats">
<div class="stat-card"><div class="val" id="totalSent">0</div><div class="lbl">Total Sent</div></div>
<div class="stat-card"><div class="val" id="totalNC">0</div><div class="lbl">NC Changes</div></div>
<div class="stat-card"><div class="val" id="activeAcc">0/6</div><div class="lbl">Active Accounts</div></div>
<div class="stat-card"><div class="val" id="pingCount">0</div><div class="lbl">Self Pings</div></div>
</div>

<div class="grid">
<div class="card system-card">
<div class="card-head">⚙️ SYSTEM LIVE LOGS <span class="badge badge-ok" id="sysBadge">LIVE</span></div>
<div class="logs" id="systemLogs"></div>
</div>
</div>

<div class="grid" id="accGrid" style="margin-top:12px"></div>

<script>
let pingCount=0, sentCount=0, ncCount=0;
function formatLogs(logs){
 return logs.slice().reverse().map(l=>{
   let cls='log-info';
   if(l.includes('✅')||l.includes('sent to')||l.includes('changed title')){cls='log-ok'; if(l.includes('sent to')) sentCount++;}
   if(l.includes('❌')||l.includes('failed')) cls='log-err';
   if(l.includes('⚠')||l.includes('cooling')) cls='log-warn';
   if(l.includes('Self ping')||l.includes('Keepalive')){cls='log-ping'; if(l.includes('Self ping')){pingCount++; document.getElementById('lastPing').innerText=new Date().toLocaleTimeString();}}
   if(l.includes('changed title')) ncCount++;
   return `<div class="${cls}">${l}</div>`;
 }).join('');
}
async function fetchAll(){
 try{
  let res=await fetch('/api/logs');
  let data=await res.json();
  // System
  document.getElementById('systemLogs').innerHTML=formatLogs(data.system||[]);
  // Uptime
  let up=Math.floor((Date.now()/1000)-data.start_time);
  let h=Math.floor(up/3600), m=Math.floor((up%3600)/60), s=up%60;
  document.getElementById('uptime').innerText=`${h}h ${m}m ${s}s`;
  document.getElementById('pingInterval').innerText=data.self_ping_interval;
  document.getElementById('keepAlive').innerText=data.keepalive_interval;
  document.getElementById('pingCount').innerText=pingCount;
  document.getElementById('totalSent').innerText=sentCount;
  document.getElementById('totalNC').innerText=ncCount;
  document.getElementById('activeAcc').innerText=`${data.active_count}/6`;
  
  // Accounts grid
  let grid=document.getElementById('accGrid');
  if(grid.children.length===0){
   for(let i=1;i<=6;i++){
     let acc=`acc${i}`;
     grid.innerHTML+=`
     <div class="card">
       <div class="card-head">${acc.toUpperCase()} <span class="badge badge-ok" id="badge-${acc}">LIVE</span></div>
       <div class="logs" id="logs-${acc}"></div>
     </div>`;
   }
  }
  for(let i=1;i<=6;i++){
    let acc=`acc${i}`;
    let el=document.getElementById(`logs-${acc}`);
    if(el) el.innerHTML=formatLogs(data[acc]||[]);
    let badge=document.getElementById(`badge-${acc}`);
    if(data[acc] && data[acc].length>0){
      let last=data[acc][data[acc].length-1];
      if(last.includes('❌')||last.includes('inactive')){badge.className='badge badge-err'; badge.innerText='ERROR';}
      else if(last.includes('cooling')||last.includes('⚠')){badge.className='badge badge-warn'; badge.innerText='COOLDOWN';}
      else{badge.className='badge badge-ok'; badge.innerText='LIVE';}
    }
  }
 }catch(e){console.log(e)}
}
setInterval(fetchAll,1500);
setInterval(()=>{document.getElementById('clock').innerText=new Date().toLocaleTimeString();},1000);
fetchAll();
</script>
</body>
</html>
'''

@app.route("/")
def dashboard():
    return render_template_string(DASHBOARD_HTML)

@app.route("/health")
def health(): return jsonify({"status": "ok"})

@app.route("/api/logs")
def api_logs():
    with logs_lock:
        data = {k: v[-100:] for k,v in session_logs.items()}
    active = sum(1 for k in ["acc1","acc2","acc3","acc4","acc5","acc6"] if any("Logged in" in l or "sent to" in l for l in session_logs.get(k,[])[-5:]))
    data["start_time"] = START_TIME
    data["self_ping_interval"] = SELF_PING_INTERVAL
    data["keepalive_interval"] = SESSION_KEEPALIVE_INTERVAL
    data["active_count"] = active
    return jsonify(data)

@app.route("/status")
def status():
    with logs_lock:
        return jsonify({k: v[-20:] for k,v in session_logs.items()})

def decode_session(s):
    if not s: return s
    try: return urllib.parse.unquote(s)
    except: return s

SETTINGS_DIR = "/tmp/settings"
os.makedirs(SETTINGS_DIR, exist_ok=True)

def get_settings_path(acc_name): return os.path.join(SETTINGS_DIR, f"settings_{acc_name}.json")
def get_ua_path(acc_name): return os.path.join(SETTINGS_DIR, f"ua_{acc_name}.txt")

def get_stable_user_agent(acc_name):
    ua_path = get_ua_path(acc_name)
    if os.path.exists(ua_path):
        with open(ua_path, "r") as f: return f.read().strip()
    idx = int(acc_name.replace("acc","")) - 1
    ua = STABLE_USER_AGENTS[idx % len(STABLE_USER_AGENTS)]
    with open(ua_path, "w") as f: f.write(ua)
    return ua

def get_stable_device_id(acc_name, session_id):
    base = f"{acc_name}_{session_id[:20]}"
    return "android-" + hashlib.md5(base.encode()).hexdigest()[:16]

def login_session(session_id, name_hint=""):
    session_id = decode_session(session_id)
    acc_name = name_hint or "system"
    settings_path = get_settings_path(acc_name)
    ua = get_stable_user_agent(acc_name)
    device_id = get_stable_device_id(acc_name, session_id)
    cl = Client()
    cl.set_user_agent(ua)
    cl.device_id = device_id
    proxy = PROXIES.get(acc_name, "")
    if proxy:
        cl.set_proxy(proxy)
    if os.path.exists(settings_path):
        try:
            cl.load_settings(settings_path)
            cl.set_user_agent(ua)
            cl.device_id = device_id
            log(f"📂 Loaded & Locked device for {acc_name}", session=acc_name)
        except Exception as e:
            log(f"⚠ Failed load {acc_name}: {e}", session=acc_name)
    try:
        cl.login_by_sessionid(session_id)
        cl.dump_settings(settings_path)
        log(f"✅ Logged in {getattr(cl,'username','?')} | UA Locked | Device {device_id}", session=acc_name)
        return cl
    except Exception as e:
        log(f"❌ Login failed {acc_name}: {e}", session=acc_name)
        return None

def is_client_alive(cl):
    try:
        cl.get_timeline_feed()
        return True
    except (LoginRequired, ChallengeRequired):
        return False
    except Exception:
        return True

def ensure_login(acc, raw_sid):
    cl = acc.get("client")
    if cl and is_client_alive(cl): return cl
    log(f"♻️ {acc['name']} re-login...", session=acc["name"])
    new_cl = login_session(raw_sid, acc["name"])
    if new_cl:
        acc["client"] = new_cl
        acc["active"] = True
        acc["cooldown_until"] = 0
        return new_cl
    acc["active"] = False
    return None

def safe_send_message(cl, gid, msg, acc_name):
    try:
        cl.direct_send(msg, thread_ids=[int(gid)])
        log(f"✅ {getattr(cl,'username','?')} sent to {gid}", session=acc_name)
        return True
    except LoginRequired:
        raise
    except Exception as e:
        log(f"⚠ Send failed {acc_name} -> {gid}: {e}", session=acc_name)
        return False

def safe_change_title_direct(cl, gid, new_title, acc_name):
    try:
        tt = cl.direct_thread(int(gid))
        tt.update_title(new_title)
        log(f"📝 {acc_name} changed title (direct) for {gid} -> {new_title}", session=acc_name)
        return True
    except Exception:
        pass
    try:
        headers = {
            "User-Agent": cl.user_agent if hasattr(cl, 'user_agent') else get_stable_user_agent(acc_name),
            "X-CSRFToken": CSRF_TOKEN,
            "X-Requested-With": "XMLHttpRequest",
            "Referer": f"https://www.instagram.com/direct/t/{gid}/",
        }
        cookies = {"csrftoken": CSRF_TOKEN}
        cl.private.headers.update(headers)
        cl.private.cookies.update(cookies)
        variables = {"thread_fbid": gid, "new_title": new_title}
        payload = {"doc_id": DOC_ID, "variables": json.dumps(variables)}
        resp = cl.private.post("https://www.instagram.com/api/graphql/", data=payload, timeout=10)
        result = resp.json()
        if "errors" not in result:
            log(f"📝 {acc_name} title (graphql) -> {new_title} for {gid}", session=acc_name)
            return True
        return False
    except Exception as e:
        log(f"❌ Title fail {gid}: {e}", session=acc_name)
        return False

def spam_loop(accounts, groups, raw_map):
    time.sleep(SPAM_START_OFFSET)
    idx = 0
    while True:
        acc = accounts[idx]
        acc_name = acc["name"]
        try:
            if acc.get("cooldown_until",0) > time.time():
                log(f"⏳ {acc_name} cooling down", session=acc_name)
            else:
                cl = ensure_login(acc, raw_map.get(acc_name))
                if cl:
                    for _ in range(BURST_COUNT):
                        for gid in groups:
                            if not safe_send_message(cl, gid, MESSAGE_TEXT, acc_name):
                                acc["cooldown_until"] = time.time() + COOLDOWN_ON_ERROR
                                break
                            time.sleep(MSG_REFRESH_DELAY + random.uniform(0.3,1.0))
        except LoginRequired:
            acc["client"]=None; acc["active"]=False
        except Exception as e:
            log(f"❌ {acc_name} spam err: {e}", session=acc_name)
            acc["cooldown_until"]=time.time()+COOLDOWN_ON_ERROR
        time.sleep(SPAM_GAP_BETWEEN_ACCOUNTS)
        idx=(idx+1)%len(accounts)

def parse_nc_titles():
    base=[t.strip() for t in NC_TITLES_RAW.split(",") if t.strip()]
    default=MESSAGE_TEXT[:40] or "NC"
    while len(base)<6: base.append(default)
    return base[:6]

def nc_loop(accounts, groups, titles_map, raw_map):
    if not groups: return
    per_titles=parse_nc_titles()
    log(f"NC titles per account: {per_titles}", session="system")
    time.sleep(NC_START_OFFSET)
    idx=0
    while True:
        acc=accounts[idx]; acc_name=acc["name"]
        try:
            if acc.get("cooldown_until",0) <= time.time():
                cl=ensure_login(acc, raw_map.get(acc_name))
                if cl:
                    for gid in groups:
                        t=(titles_map.get(str(gid)) or titles_map.get(int(gid)) or [per_titles[idx]])[0]
                        if not safe_change_title_direct(cl,gid,t,acc_name):
                            acc["cooldown_until"]=time.time()+COOLDOWN_ON_ERROR; break
                        time.sleep(1)
        except Exception as e:
            log(f"❌ {acc_name} nc err: {e}", session=acc_name)
            acc["cooldown_until"]=time.time()+COOLDOWN_ON_ERROR
        time.sleep(NC_ACC_GAP); idx=(idx+1)%len(accounts)

def session_keepalive_loop(accounts, raw_map):
    while True:
        time.sleep(SESSION_KEEPALIVE_INTERVAL)
        for acc in accounts:
            if not acc.get("client"): continue
            try:
                cl=acc["client"]
                if is_client_alive(cl):
                    cl.dump_settings(get_settings_path(acc["name"]))
                    log(f"💓 Keepalive OK {acc['name']} | Device Locked", session=acc["name"])
                else:
                    ensure_login(acc, raw_map.get(acc["name"]))
            except Exception as e:
                log(f"⚠ Keepalive fail {acc['name']}: {e}", session=acc["name"])

def self_ping_loop():
    while True:
        if SELF_URL:
            try:
                requests.get(SELF_URL,timeout=10)
                log(f"🔁 Self ping successful - {time.strftime('%H:%M:%S')} - Next in {SELF_PING_INTERVAL}s", session="system")
            except Exception as e:
                log(f"⚠ Self ping failed: {e}", session="system")
        else:
            log(f"🔁 Self ping tick - {time.strftime('%H:%M:%S')} (SELF_URL not set)", session="system")
        time.sleep(SELF_PING_INTERVAL)

def start_bot():
    sessions=[decode_session(x) for x in [SESSION_ID_1,SESSION_ID_2,SESSION_ID_3,SESSION_ID_4,SESSION_ID_5,SESSION_ID_6]]
    groups=[g.strip() for g in GROUP_IDS.split(",") if g.strip()]
    if not groups: log("❌ GROUP_IDS empty","system"); return
    titles_map={}
    raw=os.getenv("GROUP_TITLES","")
    if raw:
        try: titles_map=json.loads(raw)
        except: pass
    raw_map={f"acc{i+1}":s for i,s in enumerate(sessions) if s}
    accounts=[]
    for i,s in enumerate(sessions,1):
        acc_name=f"acc{i}"
        if not s:
            accounts.append({"name":acc_name,"client":None,"active":False,"cooldown_until":0}); continue
        log(f"🔐 Logging {acc_name} with LOCKED UA & Device...",session="system")
        cl=login_session(s,acc_name)
        accounts.append({"name":acc_name,"client":cl,"active":bool(cl),"cooldown_until":0})

    if not any(a["client"] for a in accounts): log("❌ No login","system"); return

    threading.Thread(target=spam_loop,args=(accounts,groups,raw_map),daemon=True).start()
    threading.Thread(target=nc_loop,args=(accounts,groups,titles_map,raw_map),daemon=True).start()
    threading.Thread(target=self_ping_loop,daemon=True).start()
    threading.Thread(target=session_keepalive_loop,args=(accounts,raw_map),daemon=True).start()
    log(f"▶ All loops started | Dashboard at / | SelfPing {SELF_PING_INTERVAL}s | UA & Device LOCKED",session="system")

def run_bot_once():
    threading.Thread(target=start_bot,daemon=True).start()
run_bot_once()

if __name__=="__main__":
    port=int(os.getenv("PORT","10000"))
    log(f"HTTP server starting on port {port} | Dashboard: /", session="system")
    app.run(host="0.0.0.0",port=port)
