import os
import json
import time
import random
import threading
from flask import Flask, jsonify
from instagrapi import Client

app = Flask(__name__)

USERNAME = os.getenv("IG_USERNAME", "acc1")
PASSWORD = os.getenv("IG_PASSWORD", "")
PROXY = os.getenv("IG_PROXY", "")

SESSION_FILE = f"session_{USERNAME}.json"

cl = Client()
cl.delay_range = [5, 12]

# Proxy - Render ke liye compulsory
if PROXY:
    cl.set_proxy(PROXY)
    print(f"[PROXY] Set")

# FIXED LOGIN - get_device hata diya hai, yehi crash kara raha tha
def login_client():
    # Purana session try karo
    if os.path.exists(SESSION_FILE):
        try:
            cl.load_settings(SESSION_FILE)
            cl.login(USERNAME, PASSWORD) # session se relogin
            # Safe check - info_stream kabhi mat marna
            cl.get_timeline_feed()
            print(f"[LOGIN] Session valid - {USERNAME}")
            return True
        except Exception as e:
            print(f"[LOGIN] Old session expired: {e}")

    # Fresh login
    try:
        print(f"[LOGIN] Fresh login for {USERNAME}...")
        # Device manually set karo, get_device() nahi hai
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
        cl.login(USERNAME, PASSWORD)
        cl.dump_settings(SESSION_FILE)
        print(f"[LOGIN] SUCCESS - {USERNAME}")
        return True
    except Exception as e:
        print(f"[ERROR] Login failed {USERNAME}: {e}")
        if "redirects" in str(e).lower() or "403" in str(e).lower():
            if os.path.exists(SESSION_FILE):
                os.remove(SESSION_FILE)
        return False

IS_LOGGED_IN = False

def keepalive_loop():
    global IS_LOGGED_IN
    IS_LOGGED_IN = login_client()
    while True:
        if IS_LOGGED_IN:
            try:
                cl.get_timeline_feed()
                print(f"[{time.strftime('%H:%M:%S')}] OK - {USERNAME} SAFE")
            except Exception as e:
                print(f"[LOOP] Error: {e}")
                IS_LOGGED_IN = False
                login_client()
        time.sleep(900) # 15 min - isse kam mat karna warna logout hogi

@app.route('/')
def home():
    return jsonify({"status": "running", "account": USERNAME, "logged_in": IS_LOGGED_IN})

@app.route('/api/logs')
def api_logs():
    # Yahan IG ka koi call nahi - isiliye pehle logout ho rahi thi
    return jsonify({"status": "alive", "logged_in": IS_LOGGED_IN}), 200

if __name__ == "__main__":
    threading.Thread(target=keepalive_loop, daemon=True).start()
    port = int(os.environ.get("PORT", 10000))
    app.run(host='0.0.0.0', port=port)
