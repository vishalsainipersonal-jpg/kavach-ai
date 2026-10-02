# kavach_ai.py - The Ultimate Investor Protection Shield
# Combined: Link Scanner, Brand Impersonation, Gaming App Scam, Voice Support
# Usage: python kavach_ai.py

import re
import socket
import ssl
import datetime
import subprocess
import platform
import sys
import json
import numpy as np
from urllib.parse import urlparse

# Optional libraries - install if needed
try:
    import requests
    import whois
    IMPORT_OK = True
except ImportError:
    IMPORT_OK = False

# ===================== LANGUAGE SETTINGS =====================
LANG = "hindi"  # Default: hindi, can change to english

def speak(text, lang="hi"):
    """Voice output using espeak/say on system, fallback to print."""
    if lang == "hi":
        text_tts = text
    else:
        text_tts = text
    try:
        if platform.system() == "Linux":
            subprocess.run(["espeak", "-v", "hi", text_tts], check=False)
        elif platform.system() == "Darwin":
            subprocess.run(["say", text_tts], check=False)
        else:
            print("[VOICE] " + text_tts)  # fallback
    except Exception:
        print("[VOICE] " + text_tts)
    print(text_tts)

# ===================== BRAND LIST =====================
BRANDS = ["SBI", "TATA", "RELIANCE", "ICICI", "HDFC", "AXIS", "KOTAK", "PAYTM", "PHONEPE", "GOOGLEPAY", "AMAZON", "FLIPKART", "ZOMATO", "SWIGGY", "LIC", "IRCTC", "MTNL", "BSNL"]

# ===================== 1. LINK SCANNER =====================
def scan_url(url):
    warnings = []
    parsed = urlparse(url)
    domain = parsed.netloc or parsed.path.split('/')[0]
    ip = ''
    domain_age = None
    ssl_valid = None
    hosting_country = None

    # Basic IP extraction
    try:
        ip = socket.gethostbyname(domain)
        warnings.append(f"IP Address: {ip}")
    except:
        warnings.append("IP Resolution Failed")

    # Domain age using whois if available
    if IMPORT_OK and 'whois' in sys.modules:
        try:
            w = whois.whois(domain)
            creation = w.creation_date
            if isinstance(creation, list):
                creation = creation[0]
            if creation:
                age = (datetime.datetime.now() - creation).days
                domain_age = age
                if age < 365:
                    warnings.append(f"⚠️ Domain sirf {age} din purana hai - Frauds yahan common hain!")
                else:
                    warnings.append(f"Domain {age} din purana hai (OK)")
        except:
            warnings.append("Domain age verification failed")
    else:
        warnings.append("Whois library not available, domain age check skipped")

    # SSL check
    try:
        ctx = ssl.create_default_context()
        with ctx.wrap_socket(socket.socket(), server_hostname=domain) as s:
            s.settimeout(3)
            s.connect((domain, 443))
            cert = s.getpeercert()
            ssl_valid = True
            warnings.append("SSL Certificate Valid")
    except:
        ssl_valid = False
        warnings.append("⚠️ SSL Certificate Not Valid - Fake site ho sakti hai")

    # Hosting country (simplified - through IP geolocation API if requests available)
    if IMPORT_OK and 'requests' in sys.modules:
        try:
            resp = requests.get(f"http://ip-api.com/json/{ip}", timeout=3)
            data = resp.json()
            hosting_country = data.get('country', 'Unknown')
            if hosting_country.lower() not in ['india', 'in']:
                warnings.append(f"🌍 Hosting Country: {hosting_country} - foreign hosting suspicious")
            else:
                warnings.append(f"🌍 Hosting Country: {hosting_country}")
        except:
            warnings.append("Hosting country check failed")

    # Basic heuristic: check for common fraud keywords in URL
    fraud_keywords = ['win', 'reward', 'bonus', 'free', 'lucky', 'lottery', 'gift', 'prize', 'casino', 'bitcoin', 'cryptocurrency', 'investment', 'profit', 'guaranteed', 'millionaire', 'jackpot']
    lower_url = url.lower()
    for kw in fraud_keywords:
        if kw in lower_url:
            warnings.append(f"⚠️ URL mein suspicious keyword: '{kw}'")

    return warnings

# ===================== 2. BRAND IMPERSONATION DETECTION =====================
def levenshtein(s1, s2):
    if len(s1) < len(s2):
        return levenshtein(s2, s1)
    if len(s2) == 0:
        return len(s1)
    previous_row = list(range(len(s2) + 1))
    for i, c1 in enumerate(s1):
        current_row = [i + 1]
        for j, c2 in enumerate(s2):
            insertions = previous_row[j + 1] + 1
            deletions = current_row[j] + 1
            substitutions = previous_row[j] + (c1 != c2)
            current_row.append(min(insertions, deletions, substitutions))
        previous_row = current_row
    return previous_row[-1]

def check_brand_impersonation(input_text):
    """Check if input_text resembles any known brand in a fraudulent way."""
    input_text = re.sub(r'[^a-zA-Z0-9]', '', input_text.lower())
    for brand in BRANDS:
        brand_clean = re.sub(r'[^a-zA-Z0-9]', '', brand.lower())
        # Exact match
        if brand_clean == input_text:
            return False  # legit
        # Levenshtein similarity - if edit distance small and not exact, it's impersonation
        distance = levenshtein(brand_clean, input_text)
        max_len = max(len(brand_clean), len(input_text))
        if distance <= 2 and distance > 0:
            ratio = 1 - distance / max_len
            if ratio > 0.6:  # 60% similar
                return True
    return False

# ===================== 3. GAMING & CASINO APP SCAM =====================
def analyze_gaming_app(app_name, permissions, description=""):
    warnings = []
    # Check for suspicious permissions
    dangerous_permissions = ["SEND_SMS", "READ_SMS", "RECORD_AUDIO", "CAMERA", "ACCESS_FINE_LOCATION", "READ_CONTACTS", "SYSTEM_ALERT_WINDOW"]
    for perm in dangerous_permissions:
        if perm in permissions.upper():
            warnings.append(f"⚠️ Dangerous permission: {perm}")
    
    # Check for "guaranteed profit" or similar keywords
    scam_words = ["guaranteed profit", "double your money", "instant withdrawal", "no loss", "100% return", "risk free", "high returns", "urgent", "daily income"]
    lower_desc = description.lower()
    for word in scam_words:
        if word in lower_desc:
            warnings.append(f"⚠️ Fraudulent claim: '{word}'")
    
    # Check for absence of license references (simplified)
    if "sebi" not in lower_desc and "rbi" not in lower_desc and "license" not in lower_desc:
        warnings.append("⚠️ No regulatory license mentioned (SEBI/RBI) - high risk")
    
    # Heuristic: if app name has 'win' or 'casino' or 'lottery'
    if any(x in app_name.lower() for x in ['win', 'casino', 'lottery', 'bet']):
        warnings.append("⚠️ App name suggests gambling/casino activity")
    
    if not warnings:
        return "✅ No obvious scam signals detected."
    else:
        return "\n".join(warnings)

# ===================== 4. VOICE & REGIONAL LANGUAGE =====================
def select_language():
    global LANG
    while True:
        choice = input("Choose language (1. हिंदी  2. English) : ").strip()
        if choice == "1":
            LANG = "hindi"
            speak("आपकी भाषा हिंदी हो गई है। कृपया अपना विकल्प चुनें।", lang="hi")
            break
        elif choice == "2":
            LANG = "english"
            speak("Your language is set to English. Please choose an option.", lang="en")
            break
        else:
            print("Invalid choice. Try again.")

def regional_translate(warning, lang):
    """Translate common warnings to Hindi if needed. Simplified mapping."""
    if lang == "hindi":
        translations = {
            "⚠️ Domain sirf": "चेतावनी: डोमेन केवल",
            "din purana hai": "दिन पुराना है",
            "IP Address": "आईपी पता",
            "Hosting Country": "होस्टिंग देश",
            "SSL Certificate Valid": "एसएसएल प्रमाणपत्र सही है",
            "SSL Certificate Not Valid": "एसएसएल प्रमाणपत्र सही नहीं है",
            "Fake site ho sakti hai": "नकली वेबसाइट हो सकती है",
            "URL mein suspicious keyword": "यूआरएल में संदिग्ध कीवर्ड",
            "Dangerous permission": "खतरनाक अनुमति",
            "Fraudulent claim": "धोखाधड़ी का दावा",
            "No regulatory license": "कोई नियामक लाइसेंस नहीं",
            "App name suggests": "ऐप का नाम संकेत देता है",
            "foreign hosting suspicious": "विदेशी होस्टिंग संदिग्ध है",
            "Domain age verification": "डोमेन आयु सत्यापन",
            "No obvious scam signals": "कोई स्पष्ट धोखाधड़ी संकेत नहीं"
        }
        for eng, hindi in translations.items():
            warning = warning.replace(eng, hindi)
    return warning

# ===================== MAIN CLI =====================
def main():
    if platform.system() == "Linux" and not IMPORT_OK:
        print("Note: For full features, install 'requests' and 'python-whois' using pip.")
    
    print("\n🛡️  Welcome to Kavach.AI - The Ultimate Investor Protection Shield")
    print("==================================================")
    select_language()
    
    while True:
        print("\nChoose an option:")
        if LANG == "hindi":
            print("1. लिंक/URL स्कैन करें")
            print("2. ब्रांड प्रतिरूपण जाँच (impersonation)")
            print("3. गेमिंग/कैसीनो ऐप स्कैन")
            print("4. बाहर निकलें")
        else:
            print("1. Scan URL/Link")
            print("2. Check Brand Impersonation")
            print("3. Gaming App Scan")
            print("4. Exit")
        
        choice = input("> ").strip()
        
        if choice == "1":
            url = input("Enter URL (full): ").strip()
            if not url.startswith("http"):
                url = "http://" + url
            print("\n🔍 Scanning URL...")
            results = scan_url(url)
            print("\n".join(results))
            # Speak summary
            risk = "high" if any("⚠️" in r for r in results) else "low"
            speak(f"यह लिंक असुरक्षित है" if risk=="high" else "यह लिंक सुरक्षित लगता है", lang="hi")
        
        elif choice == "2":
            text = input("Enter brand name or website: ").strip()
            if check_brand_impersonation(text):
                speak("⚠️ यह एक नकली ब्रांड हो सकता है। सावधान रहें!", lang="hi")
            else:
                speak("कोई प्रतिरूपण नहीं मिला।", lang="hi")
        
        elif choice == "3":
            app_name = input("Enter app name: ").strip()
            perms = input("Enter app permissions (comma separated, if known): ").strip()
            desc = input("Enter app description (optional): ").strip()
            result = analyze_gaming_app(app_name, perms, desc)
            print(result)
            if "⚠️" in result:
                speak("यह ऐप धोखाधड़ी वाला हो सकता है!", lang="hi")
            else:
                speak("कोई स्पष्ट जोखिम नहीं।", lang="hi")
        
        elif choice == "4":
            print("Exiting... Stay safe with Kavach.AI!")
            break
        else:
            print("Invalid option. Try again.")

if __name__ == "__main__":
    main()
