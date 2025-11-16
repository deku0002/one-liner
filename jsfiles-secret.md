🔐 JavaScript Recon & Secrets Hunting Toolkit
Bug Bounty • Penetration Testing • API Discovery • Endpoint Enumeration

This repository contains a complete workflow for discovering JavaScript files, extracting hidden API endpoints, detecting leaked secrets, and analyzing client-side logic for vulnerabilities.

Designed for Bug Bounty Hunters, Penetration Testers, and Security Researchers who want to identify high-severity issues inside frontend code.

⭐ Features

🔎 Automatic JS discovery (live + historical)

⬇️ Automated JS downloader (clean file naming)

🕵️ Secret detection (tokens, API keys, JWTs, client IDs)

🌐 Endpoint enumeration (hidden routes, internal APIs)

🧹 JS deobfuscation (beautify + readable output)

📡 Includes Wayback, Katana, Hakrawler enum

🧪 Full Recon → Analyze → Exploit workflow

💥 Real bug bounty methodology (tested live)

🚀 Quick Start (Full Workflow)
1️⃣ Enumerate Subdomains
subfinder -d target.com -o subdomains.txt

2️⃣ Probe Live Hosts
cat subdomains.txt | httpx -silent -threads 50 -o alive.txt

3️⃣ Extract JavaScript URLs (Live Crawling)
cat alive.txt | hakrawler -d 2 -subs \
  | grep -Ei '\.js($|\?)' \
  | sort -u > js_urls.txt

4️⃣ Download JS Files
mkdir -p js_files
cat js_urls.txt | xargs -n1 -P30 -I% sh -c \
'curl -sL "%" -o js_files/$(echo "%" | sed "s|https\?://||; s|[/?=&:#]|_|g")'

🕵️ 5️⃣ Setup SecretFinder (Recommended)
python3 -m venv venv
source venv/bin/activate
git clone https://github.com/m4ll0k/SecretFinder.git
cd SecretFinder
pip install -r requirements.txt
cd ..

🔍 6️⃣ Scan JS Files for Secrets
mkdir -p findings
for f in js_files/*; do
  echo "[+]> scanning $f" >> findings/secretfinder_results.txt
  python3 SecretFinder/SecretFinder.py -i "$f" -o cli \
    2>/dev/null >> findings/secretfinder_results.txt
done

🌐 7️⃣ (Optional) Scan JS URLs Directly
mkdir -p findings_urls
cat js_urls.txt | while read url; do
  echo "[+]> $url" >> findings_urls/secretfinder_urls.txt
  python3 SecretFinder/SecretFinder.py -i "$url" -o cli \
    2>/dev/null >> findings_urls/secretfinder_urls.txt
done

🎯 8️⃣ High-Confidence Secret Filtering
grep -iE "api[_-]?key|access[_-]?token|secret|bearer|jwt|client[_-]?id|private_key" findings/*.txt

🕸 9️⃣ Historical JavaScript Discovery
Wayback Machine:
echo "target.com" | waybackurls | grep "\.js$" | tee js_wayback.txt

Katana (Dynamic JS Enum):
katana -u https://target.com -extr-out -o js_live.txt

📡 🔎 Endpoint Enumeration Tools
LinkFinder

(Finds hidden internal routes)

python3 LinkFinder.py -i https://target.com/script.js -o cli

SecretFinder (Advanced)
python3 SecretFinder.py -i https://target.com/app.js -e -o output.html

🧹 JS Deobfuscation / Beautification
js-beautify ugly.js > clean.js

🧵 End-to-End Workflow (Recon → Analyze → Exploit)
Step 1 — Collect JS
waybackurls target.com | grep "\.js$" | uro \
| httpx -status-code -mc 200 -silent > js_files.txt

Step 2 — Analyze Secrets
for url in $(cat js_files.txt); do
  python3 SecretFinder.py -i $url -e
done

Step 3 — Use Endpoints

Test for:

🔥 IDOR: user_id=123 → 124

🧨 XSS: innerHTML, eval()

🔓 Auth flaws: exposed client IDs, JWT misconfigs

🔐 Misconfigs: Missing CORS, leaking admin APIs

Tip: Use Burp Suite to intercept requests loaded by JS.

💥 Real Bug Bounty Case Study — $10,000 Payout

A .js.map file leaked:

JWT signing secret

Internal admin routes

Ability to forge admin tokens

Impact:

Full account takeover

Delete/modify user data

Persistent access

Lesson:

.map files = uncompiled source code = goldmine.

🎯 Pro Tips for JS Bug Hunting

✔ Always inspect:

app-*.js

framework-*.js

webpack-runtime-*.js

sw.js / service workers

.js.map files

✔ Look for:

API base URLs

Tokens

Client IDs

Hidden routes

Sentry DSNs

Keycloak / OAuth configs

✔ Manual review catches what tools miss.

📦 Want an Automation Script?

I can generate:

run_jssecret.sh
A single command script that runs the entire workflow automatically.

Just ask: "generate the bash script"

⭐ Support This Repo

If this helped your bug bounty workflow:

⭐ Star this repository

🔗 Share with other hunters

🐞 Contribute improvements
