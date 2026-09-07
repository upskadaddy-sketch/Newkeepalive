
import os, time, threading, urllib.parse, requests, json, random
from datetime import datetime
from flask import Flask, jsonify, render_template_string
from instagrapi import Client

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
SELF_PING_INTERVAL = max(60, int(os.getenv("SELF_PING_INTERVAL", "60")))
COOLDOWN_ON_ERROR = int(os.getenv("COOLDOWN_ON_ERROR", "300"))
DOC_ID = os.getenv("DOC_ID", "29088580780787855")
CSRF_TOKEN = os.getenv("CSRF_TOKEN", "")
IG_PROXY = os.getenv("IG_PROXY", "")
PROXY_1 = os.getenv("PROXY_1", IG_PROXY)
PROXY_2 = os.getenv("PROXY_2", IG_PROXY)
PROXY_3 = os.getenv("PROXY_3", IG_PROXY)
PROXY_4 = os.getenv("PROXY_4", IG_PROXY)
PROXY_5 = os.getenv("PROXY_5", IG_PROXY)
PROXY_6 = os.getenv("PROXY_6", IG_PROXY)

app = Flask(__name__)
session_logs = {"acc1": [], "acc2": [], "acc3": [], "acc4": [], "acc5": [], "acc6": [], "system": []}
logs_lock = threading.Lock()
accounts_global = []

def _push_log(session, msg):
    if session not in session_logs: session="system"
    with logs_lock:
        session_logs[session].append(f"[{datetime.now().strftime('%H:%M:%S')}] {msg}")
        if len(session_logs[session])>500: session_logs[session].pop(0)

def log(msg, session="system"):
    print(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] {msg}", flush=True)
    _push_log(session, msg)

def decode_session(s):
    if not s: return s
    try: return urllib.parse.unquote(s)
    except: return s

def get_proxy(acc):
    return {"acc1":PROXY_1,"acc2":PROXY_2,"acc3":PROXY_3,"acc4":PROXY_4,"acc5":PROXY_5,"acc6":PROXY_6}.get(acc,IG_PROXY)

def login_session(session_id, name_hint=""):
    session_id=decode_session(session_id)
    acc_name=name_hint or "system"
    proxy=get_proxy(acc_name)
    try:
        cl=Client(request_timeout=30)
        cl.delay_range=[1,3]
        if proxy:
            try: cl.set_proxy(proxy)
            except: pass
        cl.login_by_sessionid(session_id)
        try: cl.get_timeline_feed()
        except: pass
        log(f"✅ Logged in {getattr(cl,'username','?')} - ID SAFE (bypass)", session=acc_name)
        return cl
    except Exception as e:
        log(f"❌ Login failed ({acc_name}): {str(e)[:200]}", session=acc_name)
        return None

def safe_send(cl,gid,msg,acc):
    try:
        cl.direct_send(msg, thread_ids=[int(gid)])
        log(f"✅ {getattr(cl,'username','?')} sent to {gid} | {msg[:30]}", session=acc)
        return True
    except Exception as e:
        err=str(e).lower()
        log(f"⚠ Send fail {acc}->{gid}: {err[:150]}", session=acc)
        if "login_required" in err or "challenge" in err: return False
        return True

def safe_title(cl,gid,title,acc):
    try:
        cl.direct_thread(int(gid)).update_title(title)
        log(f"📝 {acc} title {gid}->{title}", session=acc)
        return True
    except:
        try:
            cl.private.headers.update({"X-CSRFToken":CSRF_TOKEN,"X-Requested-With":"XMLHttpRequest","Referer":f"https://www.instagram.com/direct/t/{gid}/"})
            resp=cl.private.post("https://www.instagram.com/api/graphql/", data={"doc_id":DOC_ID,"variables":json.dumps({"thread_fbid":gid,"new_title":title})}, timeout=10)
            if "errors" not in resp.json():
                log(f"📝 {acc} title graphql {gid}->{title}", session=acc)
                return True
        except Exception as e2:
            log(f"❌ Title fail {gid}: {e2}", session=acc)
    return False

def spam_loop(accounts,groups):
    time.sleep(SPAM_START_OFFSET)
    idx=0
    while True:
        if not accounts: time.sleep(5); continue
        acc=accounts[idx%len(accounts)]
        try:
            if acc.get("cooldown_until",0)>time.time():
                log(f"⏳ {acc['name']} cooldown", session=acc["name"])
            elif acc["active"] and acc["client"]:
                cl=acc["client"]
                for _ in range(BURST_COUNT):
                    for gid in groups:
                        ok=safe_send(cl,gid,MESSAGE_TEXT,acc["name"])
                        if not ok:
                            new_cl=login_session(acc["raw_session"],acc["name"])
                            if new_cl: acc["client"]=new_cl
                            else: acc["active"]=False; acc["cooldown_until"]=time.time()+COOLDOWN_ON_ERROR
                            break
                        time.sleep(MSG_REFRESH_DELAY+random.uniform(0,2))
        except Exception as e:
            log(f"❌ msg loop {acc['name']}: {e}", session=acc["name"])
            acc["cooldown_until"]=time.time()+COOLDOWN_ON_ERROR
        time.sleep(SPAM_GAP_BETWEEN_ACCOUNTS+random.uniform(0,2))
        idx+=1

def parse_nc():
    base=[t.strip() for t in NC_TITLES_RAW.split(",") if t.strip()]
    default=MESSAGE_TEXT[:40] or "NC"
    while len(base)<6: base.append(default)
    return base[:6]

def nc_loop(accounts,groups,titles_map):
    time.sleep(NC_START_OFFSET)
    titles=parse_nc()
    log(f"NC titles {titles}", session="system")
    idx=0
    while True:
        if not accounts: time.sleep(5); continue
        acc=accounts[idx%len(accounts)]
        try:
            if acc["active"] and acc["client"]:
                for gid in groups:
                    t=titles_map.get(str(gid)) or [titles[idx%len(titles)]]
                    if isinstance(t,list): t=t[0]
                    safe_title(acc["client"],gid,t,acc["name"])
                    time.sleep(1)
        except Exception as e:
            log(f"❌ nc {acc['name']}: {e}", session=acc["name"])
        time.sleep(NC_ACC_GAP)
        idx+=1

def self_ping_loop():
    while True:
        if SELF_URL:
            try:
                requests.get(SELF_URL, timeout=10)
                log(f"🔁 Self ping successful - {SELF_URL}", session="system")
            except Exception as e:
                log(f"⚠ Self ping failed: {e}", session="system")
        time.sleep(SELF_PING_INTERVAL)

@app.route("/health")
def health(): return jsonify({"status":"ok","time":datetime.now().isoformat()})
@app.route("/ping")
def ping(): return "pong",200

def summarize(lines):
    rev=list(reversed(lines))
    return {"last_login": next((l for l in rev if "Logged in" in l), None), "last_send_ok": next((l for l in rev if "✅" in l and "sent to" in l), None), "last_send_error": next((l for l in rev if "Send fail" in l), None), "last_title_ok": next((l for l in rev if "📝" in l), None), "total": len(lines)}

@app.route("/status")
def status():
    with logs_lock:
        return jsonify({k: summarize(v) for k,v in session_logs.items()} | {"server_time": datetime.now().strftime("%Y-%m-%d %H:%M:%S")})

HTML="""<!doctype html><html><head><title>FINAL FIXED</title><meta name="viewport" content="width=device-width,initial-scale=1"><meta http-equiv="refresh" content="10">
<style>body{background:#0a0a0a;color:#eee;font-family:monospace;padding:12px}.card{background:#161616;border:1px solid #222;padding:12px;border-radius:12px;margin:10px 0}.green{color:#0f0}.red{color:#f55}.log{font-size:11px;white-space:pre-wrap;background:#000;padding:6px;border-radius:6px;max-height:200px;overflow:auto}.badge{padding:2px 8px;border-radius:10px;font-size:11px}.ok{background:#0f0;color:#000}.fail{background:#f00;color:#fff}a{color:#0af}</style></head><body>
<h2>🚀 FINAL - version_code BYPASS FIXED</h2>
<p>Time: {{now}} | Self: {{self_url}} | Ping: {{ping_interval}}s | <a href="/status">/status</a> | <a href="/health">/health</a> | <a href="/ping">/ping</a></p>
<div class="card"><b>Config:</b> Groups={{group_count}} | Msg="{{msg_text}}" | Gap={{spam_gap}}s NC_Gap={{nc_gap}}s | Proxy={{'SET' if proxy_set else 'NOT SET'}}<br><small>Fix: BYPASS version_code (no set_device), self ping same, dashboard with time/date</small></div>
{% for acc in accounts %}
<div class="card"><h3>{{acc.name}} - <span class="{{'green' if acc.active else 'red'}}">{{'ACTIVE SAFE ✅' if acc.active else 'INACTIVE ❌'}}</span> {% if acc.active %}<span class="badge ok">NO LOGOUT</span>{% else %}<span class="badge fail">LOGIN FAIL</span>{% endif %}</h3>
<div style="display:grid;grid-template-columns:1fr 1fr;gap:8px"><div><b>Last Login (time/date)</b><div class="log">{{acc.summary.last_login or 'none'}}</div><b>Last Send OK</b><div class="log green">{{acc.summary.last_send_ok or 'none'}}</div><b>Send Err</b><div class="log red">{{acc.summary.last_send_error or 'none'}}</div></div><div><b>NC OK</b><div class="log">{{acc.summary.last_title_ok or 'none'}}</div><b>Logs</b><div class="log">{{acc.logs_tail}}</div></div></div></div>
{% endfor %}
<div class="card"><h3>System - Self Ping / Msg / NC Times with Date</h3><div class="log">{{system_logs}}</div></div>
</body></html>
"""

@app.route("/")
@app.route("/dashboard")
def dashboard():
    with logs_lock:
        sys_logs="\n".join(session_logs["system"][-150:])
        acc_data=[]
        for i in range(1,7):
            name=f"acc{i}"
            logs=session_logs[name]
            acc_obj=next((a for a in accounts_global if a["name"]==name), {"name":name,"active":False,"client":None,"cooldown_until":0})
            acc_data.append({"name":name,"active":acc_obj.get("active",False),"cooldown_until":acc_obj.get("cooldown_until",0),"cooldown_fmt": datetime.fromtimestamp(acc_obj.get("cooldown_until",0)).strftime("%H:%M:%S") if acc_obj.get("cooldown_until",0)>time.time() else "None","proxy_status":"SET" if get_proxy(name) else "NOT SET","summary":summarize(logs),"logs_tail":"\n".join(logs[-20:])})
    return render_template_string(HTML, now=datetime.now().strftime("%Y-%m-%d %H:%M:%S"), self_url=SELF_URL or "Not Set", ping_interval=SELF_PING_INTERVAL, group_count=len([g for g in GROUP_IDS.split(",") if g.strip()]), msg_text=MESSAGE_TEXT[:40], spam_gap=SPAM_GAP_BETWEEN_ACCOUNTS, nc_gap=NC_ACC_GAP, proxy_set=bool(IG_PROXY or PROXY_1), accounts=acc_data, system_logs=sys_logs)

def start_bot():
    global accounts_global
    log(f"STARTUP: groups={GROUP_IDS[:50]}", session="system")
    sessions=[decode_session(SESSION_ID_1),decode_session(SESSION_ID_2),decode_session(SESSION_ID_3),decode_session(SESSION_ID_4),decode_session(SESSION_ID_5),decode_session(SESSION_ID_6)]
    groups=[g.strip() for g in GROUP_IDS.split(",") if g.strip()]
    if not groups: log("❌ GROUP_IDS empty", session="system"); return
    titles_map={}
    raw=os.getenv("GROUP_TITLES","")
    if raw:
        try: titles_map=json.loads(raw)
        except: pass
    accounts=[]
    for i,s in enumerate(sessions,1):
        name=f"acc{i}"
        if not s: accounts.append({"name":name,"client":None,"active":False,"cooldown_until":0,"raw_session":None}); continue
        cl=login_session(s,name)
        accounts.append({"name":name,"client":cl,"active":bool(cl),"cooldown_until":0,"raw_session":s})
    accounts_global=accounts
    if not any(a["client"] for a in accounts): log("❌ No login - SESSION_ID expired", session="system"); return
    threading.Thread(target=spam_loop, args=(accounts,groups), daemon=True).start()
    threading.Thread(target=nc_loop, args=(accounts,groups,titles_map), daemon=True).start()
    threading.Thread(target=self_ping_loop, daemon=True).start()
    log("▶ All loops started - NO LOGOUT ACTIVE", session="system")

def run_bot_once():
    threading.Thread(target=start_bot, daemon=True).start()
run_bot_once()
if __name__=="__main__":
    app.run(host="0.0.0.0", port=int(os.getenv("PORT","10000")))
