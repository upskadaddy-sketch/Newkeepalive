import os
import json
import time
import random
import threading
from flask import Flask, jsonify
from instagrapi import Client
from instagrapi.exceptions import LoginRequired, ChallengeRequired

app = Flask(__name__)

# ========== CONFIG - Render Environment se lega ==========
# Render Dashboard -> Environment me ye 3 daal de
# IG_USERNAME = teri id
# IG_PASSWORD = tera pass
# IG_PROXY = http://user:pass@ip:port  (RESIDENTIAL PROXY - Ye lagana hi padega)

USERNAME = os.getenv("IG_USERNAME", "acc1")
PASSWORD = os.getenv("IG_PASSWORD", "")
PROXY = os.getenv("IG_PROXY", "")

SESSION_FILE = f"session_{USERNAME}.json"
DEVICE_FILE = f"device_{USERNAME}.json"

# ========== CLIENT SETUP - YEHI FIX HAI ==========
cl = Client()
# Har request me 5-12 sec ka random delay - Instagram bot na samjhe
cl.delay_range = [5, 12]

# Device ID hamesha same rakho, har baar naya banoge to logout hogi
if os.path.exists(DEVICE_FILE):
    try:
        with open(DEVICE_FILE, 'r') as f:
            cl.set_device(json.load(f))
        print(f"[DEVICE] Loaded saved device for {USERNAME}")
    except:
        pass
else:
    device = cl.get_device()
    with open(DEVICE_FILE, 'w') as f:
        json.dump(device, f)
    print(f"[DEVICE] Created & saved new device for {USERNAME}")

# Proxy set karo - Render ke IP se direct login = 100% logout
if PROXY and PROXY.strip() != "":
    cl.set_proxy(PROXY)
    print(f"[PROXY] Proxy set")
else:
    print("[WARNING] PROXY nahi lagaya, isiliye 403 / 30 redirects aa raha hai!")

# ========== LOGIN LOGIC - TURANT LOGOUT FIX ==========
def login_client():
    # 1. Pehle purana session try karo - Fresh login bar bar karoge to ID dead
    if os.path.exists(SESSION_FILE):
        try:
            cl.load_settings(SESSION_FILE)
            # FIX: info_stream kabhi call mat karo, ye 403 deta hai. get_timeline_feed safe hai
            cl.get_timeline_feed()
            print(f"[LOGIN] Session valid hai - {USERNAME} login skip")
            return True
        except Exception as e:
            print(f"[LOGIN] Session expired: {e}")

    # 2. Agar session nahi hai tabhi fresh login
    try:
        print(f"[LOGIN] Fresh login try for {USERNAME}...")
        # Challenge handle karna zaruri hai warna 30 redirects loop
        cl.login(USERNAME, PASSWORD)
        cl.dump_settings(SESSION_FILE)
        print(f"[LOGIN] SUCCESS - Session saved to {SESSION_FILE}")
        return True
    except ChallengeRequired as e:
        print(f"[ERROR] ChallengeRequired - Phone pe ja ke 'This was me' karna padega: {e}")
        print(f"[ERROR] Ya email pe code aaya hoga")
        return False
    except Exception as e:
        err = str(e).lower()
        if "30 redirects" in err or "403" in err or "forbidden" in err:
            print(f"[ERROR] Instagram ne IP block kiya hai: {e}")
            print("[SOLUTION] 1. Residential Proxy lagao 2. 10 min wait karo 3. session.json delete karo")
            if os.path.exists(SESSION_FILE):
                os.remove(SESSION_FILE)
        else:
            print(f"[ERROR] Login failed: {e}")
        return False

# Global flag
IS_LOGGED_IN = False

def keepalive_loop():
    global IS_LOGGED_IN
    # Pehle ek baar login
    IS_LOGGED_IN = login_client()
    if not IS_LOGGED_IN:
        print("[FATAL] Login hi nahi hua, loop band")
        return

    # FIX: Instagram ko har second hit mat karo, isiliye logout ho rahi thi
    # Pehle tu har 2 sec me call kar raha tha
    while True:
        try:
            # Sirf safe endpoint - info_stream / user_info kabhi nahi
            cl.get_timeline_feed()
            print(f"[{time.strftime('%H:%M:%S')}] Keepalive OK - ID SAFE hai - {USERNAME}")
            IS_LOGGED_IN = True
        except Exception as e:
            print(f"[LOOP ERROR] {e}")
            IS_LOGGED_IN = False
            # Session dead hai to wapas login
            login_client()
        
        # 15 MINUTE WAIT - YE SABSE IMPORTANT FIX HAI
        # Isse kam karoge to turant logout hogi
        time.sleep(900)  # 900 sec = 15 min

# ========== FLASK ROUTES - ISME IG CALL MAT KARNA ==========
# Tera /api/logs har 2 sec me hit ho raha tha (screenshot me dikh raha hai)
# Agar is route ke andar IG call kiya to 100% logout hogi

@app.route('/')
def home():
    return jsonify({"status": "running", "account": USERNAME, "logged_in": IS_LOGGED_IN})

@app.route('/api/logs')
def api_logs():
    # YAHAN PE KABHI BHI cl.user_info() ya cl.get... mat lagana
    # Ye sirf Render ko batane ke liye hai ki service alive hai
    return jsonify({
        "status": "alive",
        "account": USERNAME,
        "logged_in": IS_LOGGED_IN,
        "message": "ID logout fix applied - no instagram call here"
    }), 200

# ========== START ==========
if __name__ == "__main__":
    # Instagram wala kaam background thread me
    t = threading.Thread(target=keepalive_loop, daemon=True)
    t.start()
    
    # Flask main thread pe
    port = int(os.environ.get("PORT", 10000))
    app.run(host='0.0.0.0', port=port, debug=False)
