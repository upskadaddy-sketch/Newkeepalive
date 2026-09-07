import os
import time
import threading
import urllib.parse
import requests
import json
import random
from datetime import datetime
from flask import Flask, jsonify, render_template_string
from instagrapi import Client
from instagrapi.exceptions import LoginRequired, ChallengeRequired, PleaseWaitFewMinutes

# --------- CONFIG (via env) SAME AS YOUR ORIGINAL ----------
SESSION_ID_1 = os.getenv("SESSION_ID_1")
SESSION_ID_2 = os.getenv("SESSION_ID_2")
SESSION_ID_3 = os.getenv("SESSION_ID_3")
SESSION_ID_4 = os.getenv("SESSION_ID_4")
SESSION_ID_5 = os.getenv("SESSION_ID_5")
SESSION_ID_6 = os.getenv("SESSION_ID_6")
GROUP_IDS = os.getenv("GROUP_IDS", "")
MESSAGE_TEXT = os.getenv("MESSAGE_TEXT", "Hello 👋")
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
MAX_SESSION_LOGS = 500
session_logs = {"acc1": [], "acc2": [], "acc3": [], "acc4": [], "acc5": [], "acc6": [], "system": []}
logs_lock = threading.Lock()
accounts_global = []

def _push_log(session, msg):
    if session not in session_logs:
        session = "system"
    with logs_lock:
        session_logs[session].append(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] {msg}")
        if len(session_logs[session]) > MAX_SESSION_LOGS:
            session_logs[session].pop(0)

def log(msg, session="system"):
    print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] {msg}", flush=True)
    _push_log(session, msg)

def decode_session(session):
    if not session: return session
    try: return urllib.parse.unquote(session)
    except: return session

def classify_error(e):
    msg = str(e).lower()
    if isinstance(e, (LoginRequired, ChallengeRequired)) or "login_required" in msg or "challenge" in msg or "logged out" in msg or "session expired" in msg or "401" in msg or "403" in msg or "user_has_logged_out" in msg:
        return "auth"
    if "429" in msg or "rate limit" in msg or "wait few minutes" in msg or isinstance(e, PleaseWaitFewMinutes):
        return "rate_limit"
    if "timeout" in msg or "connection" in msg or "network" in msg:
        return "network"
    return "transient"

def get_proxy_for_acc(acc_name):
    mapping = {"acc1": PROXY_1, "acc2": PROXY_2, "acc3": PROXY_3, "acc4": PROXY_4, "acc5": PROXY_5, "acc6": PROXY_6}
    return mapping.get(acc_name, IG_PROXY)

def login_session(session_id, name_hint=""):
    session_id = decode_session(session_id)
    acc_name = name_hint or "system"
    proxy = get_proxy_for_acc(acc_name)
    try:
        cl = Client()
        cl.set_device({
            "app_version": "269.0.0.18.75",
            "android_version": 26,
            "android_release": "8.0.0",
            "dpi": "480dpi",
            "resolution": "1080x1920",
            "manufacturer": "OnePlus",
            "device": "ONEPLUS A3003",
            "model": "OnePlus3",
            "cpu": "qcom"
        })
        cl.delay_range = [2, 6]
        if proxy:
            cl.set_proxy(proxy)
            log(f"🌐 Proxy set for {acc_name}", session=acc_name)
        cl.login_by_sessionid(session_id)
        cl.get_timeline_feed()
        uname = getattr(cl, "username", None) or name_hint or "unknown"
        log(f"✅ Logged in {uname} - ID SAFE", session=acc_name)
        return cl
    except Exception as e:
        err_type = classify_error(e)
        log(f"❌ Login failed ({name_hint}) [{err_type}]: {e}", session=acc_name)
        return None

def safe_send_message(cl, gid, msg, acc_name):
    for attempt in range(1, 4):
        try:
            cl.direct_send(msg, thread_ids=[int(gid)])
            log(f"✅ {getattr(cl,'username','?')} sent to {gid} | {msg[:30]}", session=acc_name)
            return True
        except Exception as e:
            err_type = classify_error(e)
            log(f"⚠ Send failed [{err_type}] att {attempt} {getattr(cl,'username','?')} -> {gid}: {e}", session=acc_name)
            if err_type == "auth": return False
            if err_type == "rate_limit":
                time.sleep(30); return False
            if attempt < 3 and err_type in ("network","transient"):
                time.sleep(2**attempt + random.random()); continue
            return False

def safe_change_title_direct(cl, gid, new_title, acc_name):
    try:
        tt = cl.direct_thread(int(gid))
        try:
            tt.update_title(new_title)
            log(f"📝 {getattr(cl,'username','?')} title direct {gid} -> {new_title}", session=acc_name)
            return True
        except: pass
    except: pass
    try:
        headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)","X-CSRFToken": CSRF_TOKEN,"X-Requested-With": "XMLHttpRequest","Referer": f"https://www.instagram.com/direct/t/{gid}/"}
        cookies = {"csrftoken": CSRF_TOKEN}
        cl.private.headers.update(headers)
        cl.private.cookies.update(cookies)
        variables = {"thread_fbid": gid, "new_title": new_title}
        payload = {"doc_id": DOC_ID, "variables": json.dumps(variables)}
        resp = cl.private.post("https://www.instagram.com/api/graphql/", data=payload, timeout=10)
        result = resp.json()
        if "errors" in result:
            log(f"❌ GraphQL errors {gid}: {result['errors']}", session=acc_name)
            return False
        log(f"📝 {getattr(cl,'username','?')} title graphql {gid} -> {new_title}", session=acc_name)
        return True
    except Exception as e:
        log(f"❌ Title change fail {gid}: {e}", session=acc_name)
        return False

def spam_loop(accounts, groups):
    time.sleep(SPAM_START_OFFSET)
    idx = 0; n = len(accounts)
    if n == 0: return
    while True:
        acc = accounts[idx]
        acc_name = acc["name"]
        try:
            if acc.get("cooldown_until", 0) > time.time():
                log(f"⏳ {acc_name} cooldown till {datetime.fromtimestamp(acc['cooldown_until']).strftime('%H:%M:%S')}", session=acc_name)
            elif not acc["active"] or not acc["client"]:
                log(f"⏭ {acc_name} inactive", session=acc_name)
            else:
                cl = acc["client"]
                for _ in range(BURST_COUNT):
                    for gid in groups:
                        ok = safe_send_message(cl, gid, MESSAGE_TEXT, acc_name)
                        if not ok:
                            log(f"🔄 {acc_name} trying re-login once...", session=acc_name)
                            new_cl = login_session(acc["raw_session"], acc_name)
                            if new_cl:
                                acc["client"] = new_cl
                                log(f"♻️ {acc_name} re-login OK", session=acc_name)
                            else:
                                acc["active"] = False
                                log(f"⛔ {acc_name} inactive need new SESSION_ID", session=acc_name)
                                break
                        time.sleep(MSG_REFRESH_DELAY + random.uniform(0,1))
        except Exception as e:
            log(f"❌ Exception {acc_name} msg loop: {e}", session=acc_name)
            acc["cooldown_until"] = time.time() + COOLDOWN_ON_ERROR
        time.sleep(SPAM_GAP_BETWEEN_ACCOUNTS + random.uniform(0,2))
        idx = (idx + 1) % n

def parse_nc_titles():
    base = [t.strip() for t in NC_TITLES_RAW.split(",") if t.strip()]
    default_title = MESSAGE_TEXT[:40] or "NC"
    while len(base) < 6: base.append(default_title)
    return base[:6]

def nc_loop(accounts, groups, titles_map):
    if not groups:
        log("⚠ No groups for title loop.", session="system"); return
    per_account_titles = parse_nc_titles()
    log(f"NC titles: {per_account_titles}", session="system")
    time.sleep(NC_START_OFFSET)
    idx = 0; n = len(accounts)
    while True:
        acc = accounts[idx]
        acc_name = acc["name"]
        account_title = per_account_titles[idx]
        try:
            if acc.get("cooldown_until", 0) > time.time():
                log(f"⏳ {acc_name} cooldown", session=acc_name)
            elif not acc["active"] or not acc["client"]:
                log(f"⏭ {acc_name} inactive nc", session=acc_name)
            else:
                cl = acc["client"]
                for gid in groups:
                    titles = titles_map.get(str(gid)) or titles_map.get(int(gid)) or [account_title]
                    t = titles[0]
                    ok = safe_change_title_direct(cl, gid, t, acc_name)
                    if not ok:
                        acc["cooldown_until"] = time.time() + COOLDOWN_ON_ERROR
                        break
                    time.sleep(1)
        except Exception as e:
            log(f"❌ Exception {acc_name} nc: {e}", session=acc_name)
            acc["cooldown_until"] = time.time() + COOLDOWN_ON_ERROR
        time.sleep(NC_ACC_GAP)
        idx = (idx + 1) % n

def self_ping_loop():
    while True:
        if SELF_URL:
            try:
                requests.get(SELF_URL, timeout=10)
                log(f"🔁 Self ping OK {SELF_URL}", session="system")
            except Exception as e:
                log(f"⚠ Ping fail: {e}", session="system")
        time.sleep(SELF_PING_INTERVAL)

@app.route("/health")
def health():
    return jsonify({"status": "ok", "time": datetime.now().isoformat()})

@app.route("/ping")
def ping():
    return "pong", 200

def summarize(lines):
    rev = list(reversed(lines))
    last_login = next((l for l in rev if "Logged in" in l), None)
    last_send_ok = next((l for l in rev if "✅" in l and "sent to" in l), None)
    last_send_err = next((l for l in rev if "Send failed" in l), None)
    last_title_ok = next((l for l in rev if "changed title" in l and "📝" in l), None)
    last_title_err = next((l for l in rev if "GraphQL title" in l or "Title change" in l), None)
    return {"last_login": last_login, "last_send_ok": last_send_ok, "last_send_error": last_send_err, "last_title_ok": last_title_ok, "last_title_error": last_title_err, "total": len(lines)}

@app.route("/status")
def status():
    with logs_lock:
        data = {k: session_logs[k][-100:] for k in session_logs}
    return jsonify({k: summarize(v) for k,v in data.items()} | {"server_time": datetime.now().strftime("%Y-%m-%d %H:%M:%S")})

DASHBOARD_HTML = """
<!doctype html>
<html><head><title>FINAL - NO LOGOUT</title>
<meta name="viewport" content="width=device-width,initial-scale=1">
<meta http-equiv="refresh" content="15">
<style>
body{background:#0a0a0a;color:#eee;font-family:monospace;padding:12px}
.card{background:#161616;border:1px solid #222;padding:12px;border-radius:12px;margin:10px 0}
.green{color:#0f0} .red{color:#f55} .yellow{color:#ff0} .blue{color:#0af}
.log{font-size:11px;white-space:pre-wrap;background:#000;padding:6px;border-radius:6px;max-height:160px;overflow:auto}
.badge{padding:2px 8px;border-radius:10px;font-size:11px} .ok{background:#0f0;color:#000} .fail{background:#f00;color:#fff} .wait{background:#ff0;color:#000}
a{color:#0af}
</style></head><body>
<h2>🚀 FINAL PANEL - Logout Fixed + Dashboard</h2>
<p>Time: {{now}} | Self: {{self_url}} | Ping: {{ping_interval}}s | <a href="/status">/status</a> | <a href="/health">/health</a></p>
<div class="card"><b>Config:</b> Groups={{group_count}} | Msg="{{msg_text}}" | SpamGap={{spam_gap}}s NC_Gap={{nc_gap}}s | Proxy={{'SET ✅' if proxy_set else 'NOT SET ❌'}}<br><small>Fix: device fixed, proxy, timeline_feed safe check, retry logic, no info_stream</small></div>
{% for acc in accounts %}
<div class="card">
<h3>{{acc.name}} - <span class="{{'green' if acc.active else 'red'}}">{{'ACTIVE SAFE' if acc.active else 'INACTIVE'}}</span> {% if acc.cooldown_until>now_ts %}<span class="badge wait">COOLDOWN {{acc.cooldown_fmt}}</span>{% elif acc.active %}<span class="badge ok">NO LOGOUT</span>{% else %}<span class="badge fail">LOGIN FAIL</span>{% endif %}</h3>
<small>Proxy: {{acc.proxy_status}} | Cooldown: {{acc.cooldown_fmt}}</small>
<div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-top:6px">
<div><b>Last Login</b><div class="log">{{acc.summary.last_login or 'none'}}</div><b>Last Send OK (time)</b><div class="log green">{{acc.summary.last_send_ok or 'none'}}</div><b>Last Send Err</b><div class="log red">{{acc.summary.last_send_error or 'none'}}</div></div>
<div><b>Last NC OK (time)</b><div class="log">{{acc.summary.last_title_ok or 'none'}}</div><b>NC Err</b><div class="log red">{{acc.summary.last_title_error or 'none'}}</div><b>Logs</b><div class="log">{{acc.logs_tail}}</div></div>
</div></div>
{% endfor %}
<div class="card"><h3>System - Ping / Msg / NC Times</h3><div class="log">{{system_logs}}</div></div>
</body></html>
"""

@app.route("/")
@app.route("/dashboard")
def dashboard():
    with logs_lock:
        sys_logs = "\n".join(session_logs["system"][-120:])
        acc_data = []
        for i in range(1,7):
            name = f"acc{i}"
            logs = session_logs[name]
            acc_obj = next((a for a in accounts_global if a["name"]==name), {"name":name,"active":False,"client":None,"cooldown_until":0})
            acc_data.append({
                "name": name, "active": acc_obj.get("active", False), "client": acc_obj.get("client"),
                "cooldown_until": acc_obj.get("cooldown_until",0),
                "cooldown_fmt": datetime.fromtimestamp(acc_obj.get("cooldown_until",0)).strftime("%H:%M:%S") if acc_obj.get("cooldown_until",0) > time.time() else "None",
                "proxy_status": "SET" if get_proxy_for_acc(name) else "NOT SET",
                "summary": summarize(logs),
                "logs_tail": "\n".join(logs[-15:])
            })
    return render_template_string(DASHBOARD_HTML, now=datetime.now().strftime("%Y-%m-%d %H:%M:%S"), now_ts=time.time(), self_url=SELF_URL or "Not Set", ping_interval=SELF_PING_INTERVAL, group_count=len([g for g in GROUP_IDS.split(",") if g.strip()]), msg_text=MESSAGE_TEXT[:40], spam_gap=SPAM_GAP_BETWEEN_ACCOUNTS, nc_gap=NC_ACC_GAP, proxy_set=bool(IG_PROXY), accounts=acc_data, system_logs=sys_logs)

def start_bot():
    global accounts_global
    log(f"STARTUP: groups={GROUP_IDS[:50]} msg={MESSAGE_TEXT[:20]} proxy_set={bool(IG_PROXY)}", session="system")
    sessions = [decode_session(SESSION_ID_1), decode_session(SESSION_ID_2), decode_session(SESSION_ID_3), decode_session(SESSION_ID_4), decode_session(SESSION_ID_5), decode_session(SESSION_ID_6)]
    groups = [g.strip() for g in GROUP_IDS.split(",") if g.strip()]
    if not groups:
        log("❌ GROUP_IDS empty", session="system"); return
    titles_map = {}
    raw_titles = os.getenv("GROUP_TITLES", "")
    if raw_titles:
        try: titles_map = json.loads(raw_titles)
        except Exception as e: log(f"⚠ GROUP_TITLES parse err: {e}", session="system")
    accounts = []
    for i, s in enumerate(sessions, 1):
        acc_name = f"acc{i}"
        if not s:
            accounts.append({"name": acc_name, "client": None, "active": False, "cooldown_until": 0, "raw_session": None})
            continue
        cl = login_session(s, acc_name)
        accounts.append({"name": acc_name, "client": cl, "active": bool(cl), "cooldown_until": 0, "raw_session": s})
    accounts_global = accounts
    if not any(a["client"] for a in accounts):
        log("❌ No accounts logged in", session="system"); return
    threading.Thread(target=spam_loop, args=(accounts, groups), daemon=True).start()
    threading.Thread(target=nc_loop, args=(accounts, groups, titles_map), daemon=True).start()
    threading.Thread(target=self_ping_loop, daemon=True).start()
    log("▶ All loops started - ID Logout Fixed", session="system")

def run_bot_once():
    threading.Thread(target=start_bot, daemon=True).start()

run_bot_once()
if __name__ == "__main__":
    port = int(os.getenv("PORT", "10000"))
    log(f"HTTP on {port}", session="system")
    app.run(host="0.0.0.0", port=port)
