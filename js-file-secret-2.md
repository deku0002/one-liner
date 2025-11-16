#!/bin/bash

# ================================================================
#   JavaScript Recon & Secrets Hunting – Full Automated Pipeline
#   Colorized | Bug Bounty Ready | Fully Terminal Friendly
#   Usage: ./run_jssecret.sh target.com
# ================================================================

# ---------- COLORS ----------
RED="\e[31m"
GREEN="\e[32m"
YELLOW="\e[33m"
BLUE="\e[34m"
CYAN="\e[36m"
BOLD="\e[1m"
RESET="\e[0m"

clear

echo -e "${GREEN}${BOLD}
██████╗ ██╗   ██╗ ██████╗      ██████╗ ███████╗ ██████╗ ██████╗ ███╗   ██╗
██╔══██╗██║   ██║██╔════╝     ██╔════╝ ██╔════╝██╔════╝██╔═══██╗████╗  ██║
██████╔╝██║   ██║██║  ███╗    ██║  ███╗█████╗  ██║     ██║   ██║██╔██╗ ██║
██╔══██╗██║   ██║██║   ██║    ██║   ██║██╔══╝  ██║     ██║   ██║██║╚██╗██║
██████╔╝╚██████╔╝╚██████╔╝    ╚██████╔╝███████╗╚██████╗╚██████╔╝██║ ╚████║
╚═════╝  ╚═════╝  ╚═════╝       ╚═════╝ ╚══════╝ ╚═════╝ ╚═════╝ ╚═╝  ╚═══╝${RESET}"

echo -e "${CYAN}${BOLD}JavaScript Recon & Secrets Hunter – Automated Bash Toolkit${RESET}\n"

# ---------- DOMAIN CHECK ----------
if [ -z "$1" ]; then
    echo -e "${RED}Usage: ./run_jssecret.sh target.com${RESET}"
    exit 1
fi

TARGET=$1

echo -e "${YELLOW}[+] Target set to:${RESET} ${GREEN}$TARGET${RESET}\n"

# ---------- CREATE DIRECTORIES ----------
mkdir -p subdomains js_files findings findings_urls wayback

# ================================================================
# 1️⃣ SUBDOMAIN ENUMERATION
# ================================================================
echo -e "${BLUE}${BOLD}[1] Subdomain Enumeration...${RESET}"
subfinder -d $TARGET -o subdomains/subdomains.txt

echo -e "${GREEN}[✓] Subdomains saved:${RESET} subdomains/subdomains.txt\n"


# ================================================================
# 2️⃣ LIVE HOST CHECK
# ================================================================
echo -e "${BLUE}${BOLD}[2] Probing Live Hosts...${RESET}"
cat subdomains/subdomains.txt | httpx -silent -threads 50 -o subdomains/alive.txt

echo -e "${GREEN}[✓] Live hosts saved:${RESET} subdomains/alive.txt\n"


# ================================================================
# 3️⃣ JS URL DISCOVERY
# ================================================================
echo -e "${BLUE}${BOLD}[3] Discovering JS URLs...${RESET}"
cat subdomains/alive.txt | hakrawler -d 2 -subs \
  | grep -Ei '\.js($|\?)' \
  | sort -u > js_urls.txt

echo -e "${GREEN}[✓] JS URLs saved:${RESET} js_urls.txt\n"


# ================================================================
# 4️⃣ DOWNLOAD JS FILES
# ================================================================
echo -e "${BLUE}${BOLD}[4] Downloading JS Files...${RESET}"

mkdir -p js_files
cat js_urls.txt | xargs -n1 -P30 -I% sh -c \
'curl -sL "%" -o js_files/$(echo "%" | sed "s|https\?://||; s|[/?=&:#]|_|g")'

echo -e "${GREEN}[✓] JS files saved to js_files/${RESET}\n"


# ================================================================
# 5️⃣ SETUP SECRETFINDER
# ================================================================
if [ ! -d "SecretFinder" ]; then
    echo -e "${BLUE}${BOLD}[5] Setting up SecretFinder...${RESET}"

    python3 -m venv venv
    source venv/bin/activate

    git clone https://github.com/m4ll0k/SecretFinder.git
    cd SecretFinder
    pip install -r requirements.txt
    cd ..
fi

echo -e "${GREEN}[✓] SecretFinder ready${RESET}\n"


# ================================================================
# 6️⃣ SCAN DOWNLOADED JS FILES
# ================================================================
echo -e "${BLUE}${BOLD}[6] Scanning JS Files for Secrets...${RESET}"

mkdir -p findings
> findings/secretfinder_results.txt

for f in js_files/*; do
  echo -e "${CYAN}[+] Scanning:${RESET} $f"
  python3 SecretFinder/SecretFinder.py -i "$f" -o cli 2>/dev/null \
    >> findings/secretfinder_results.txt
done

echo -e "${GREEN}[✓] Results saved:${RESET} findings/secretfinder_results.txt\n"


# ================================================================
# 7️⃣ OPTIONAL: URL-BASED SCAN
# ================================================================
echo -e "${BLUE}${BOLD}[7] Scanning JS URLs for Secrets...${RESET}"

mkdir -p findings_urls
> findings_urls/secretfinder_urls.txt

cat js_urls.txt | while read url; do
  echo -e "${CYAN}[+] URL:${RESET} $url"
  python3 SecretFinder/SecretFinder.py -i "$url" -o cli 2>/dev/null \
    >> findings_urls/secretfinder_urls.txt
done

echo -e "${GREEN}[✓] URL scan saved:${RESET} findings_urls/secretfinder_urls.txt\n"


# ================================================================
# 8️⃣ EXTRACT HIGH-CONFIDENCE SECRETS
# ================================================================
echo -e "${BLUE}${BOLD}[8] Extracting High-Confidence Secrets...${RESET}"

echo -e "\n${YELLOW}[!] Possible Secrets:${RESET}"
grep -iE "api[_-]?key|access[_-]?token|secret|bearer|jwt|client[_-]?id|private_key" findings/*.txt 2>/dev/null

echo -e "${GREEN}\n[✓] Secret extraction complete${RESET}\n"


# ================================================================
# 9️⃣ WAYBACK HISTORICAL JS
# ================================================================
echo -e "${BLUE}${BOLD}[9] Wayback Machine JS Discovery...${RESET}"

echo "$TARGET" | waybackurls | grep "\.js$" | tee wayback/js_wayback.txt

echo -e "${GREEN}[✓] Historical JS saved:${RESET} wayback/js_wayback.txt\n"


# ================================================================
# 🔚 FINISH
# ================================================================
echo -e "${GREEN}${BOLD}All tasks complete!${RESET}"
echo -e "${CYAN}Check your folders:${RESET}"

echo -e "
🔹 subdomains/
🔹 js_files/
🔹 findings/
🔹 findings_urls/
🔹 wayback/
🔹 js_urls.txt
"
echo -e "${GREEN}${BOLD}Happy Hunting! 🔥${RESET}\n"
