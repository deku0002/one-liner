#!/bin/bash

# -----------------------------------------------------------------------------
# 🔐 JavaScript Recon & Secrets Hunting Toolkit
#
# This script automates the full workflow for discovering and analyzing
# JavaScript files to find API endpoints and potential hardcoded secrets.
#
# Designed for Bug Bounty Hunters, Penetration Testers, and Security Researchers.
#
# -----------------------------------------------------------------------------
#
# USAGE:
#   1. Save this file as `run_js_recon.sh`
#   2. Make it executable: `chmod +x run_js_recon.sh`
#   3. Run it: `./run_js_recon.sh target.com`
#
# -----------------------------------------------------------------------------

# --- CONFIGURATION ---

# Number of parallel threads for tools that support it
THREADS=50

# Main target domain (passed as the first argument)
TARGET_DOMAIN=$1

# Directory to store all findings for this target
OUTPUT_DIR="recon_$(echo "$TARGET_DOMAIN" | sed 's/\./_/g')"

# Tool paths (optional, assumes they are in your $PATH)
SECRETFINDER_PATH="SecretFinder/SecretFinder.py"

# --- COLORS ---
RESET="\033[0m"
RED="\033[0;31m"
GREEN="\033[0;32m"
YELLOW="\033[0;33m"
BLUE="\033[0;34m"
BOLD="\033[1m"

# --- HELPER FUNCTIONS ---

# Print a formatted header
print_header() {
  echo -e "\n${BLUE}${BOLD}--- $1 ---${RESET}"
}

# Print a success message
print_success() {
  echo -e "${GREEN}[+]${RESET} $1"
}

# Print an info message
print_info() {
  echo -e "${YELLOW}[*]${RESET} $1"
}

# Print an error message and exit
print_error() {
  echo -e "${RED}[!]${RESET} $1"
  exit 1
}

# Check if a command exists
check_command() {
  if ! command -v "$1" &> /dev/null; then
    print_error "$1 could not be found. Please install it."
  fi
}

# Check if the target domain was provided
if [ -z "$TARGET_DOMAIN" ]; then
  echo "Usage: $0 <target.com>"
  exit 1
fi

# --- TOOL CHECKS ---
print_info "Checking required tools..."
check_command "subfinder"
check_command "httpx"
check_command "hakrawler"
check_command "curl"
check_command "python3"
check_command "waybackurls"
check_command "katana"
check_command "grep"
check_command "sort"
check_command "tee"
check_command "xargs"
check_command "sed"

# --- SETUP WORKSPACE ---
print_info "Creating workspace at $OUTPUT_DIR"
mkdir -p "$OUTPUT_DIR/js_files"
mkdir -p "$OUTPUT_DIR/findings"

# --- WORKFLOW ---

# 1️⃣ Enumerate Subdomains
print_header "STEP 1: ENUMERATING SUBDOMAINS"
print_info "Running subfinder..."
subfinder -d "$TARGET_DOMAIN" -o "$OUTPUT_DIR/subdomains.txt" -silent
print_success "Subdomain enumeration complete. Found $(wc -l < "$OUTPUT_DIR/subdomains.txt") subdomains."

# 2️⃣ Probe Live Hosts
print_header "STEP 2: PROBING LIVE HOSTS"
print_info "Running httpx..."
cat "$OUTPUT_DIR/subdomains.txt" | httpx -silent -threads "$THREADS" -o "$OUTPUT_DIR/alive.txt"
print_success "Live host probing complete. Found $(wc -l < "$OUTPUT_DIR/alive.txt") live hosts."

# 3️⃣ Extract JavaScript URLs (Live Crawling)
print_header "STEP 3: EXTRACTING JAVASCRIPT URLS (LIVE)"
print_info "Running hakrawler on live hosts..."
cat "$OUTPUT_DIR/alive.txt" | hakrawler -d 2 -subs |
  grep -Ei '\.js($|\?)' |
  sort -u > "$OUTPUT_DIR/js_urls_live.txt"
print_success "Live JS URL extraction complete."

# 9️⃣ Historical JavaScript Discovery
print_header "STEP 4: EXTRACTING JAVASCRIPT URLS (HISTORICAL)"
print_info "Running waybackurls..."
echo "$TARGET_DOMAIN" | waybackurls | grep -Ei '\.js($|\?)' | sort -u > "$OUTPUT_DIR/js_urls_wayback.txt"

print_info "Running katana (this may take a while)..."
katana -u "https://$TARGET_DOMAIN" -extr-out -silent -o "$OUTPUT_DIR/js_urls_katana.txt"
grep -Ei '\.js($|\?)' "$OUTPUT_DIR/js_urls_katana.txt" | sort -u > "$OUTPUT_DIR/js_urls_katana_filtered.txt"

print_info "Combining all JS URLs..."
cat "$OUTPUT_DIR/js_urls_live.txt" "$OUTPUT_DIR/js_urls_wayback.txt" "$OUTPUT_DIR/js_urls_katana_filtered.txt" |
  sort -u > "$OUTPUT_DIR/js_urls_all.txt"
print_success "Total unique JS URLs found: $(wc -l < "$OUTPUT_DIR/js_urls_all.txt")"

# 4️⃣ Download JS Files
print_header "STEP 5: DOWNLOADING JAVASCRIPT FILES"
print_info "Downloading files (using 30 parallel processes)..."
mkdir -p "$OUTPUT_DIR/js_files"
cat "$OUTPUT_DIR/js_urls_all.txt" | xargs -n1 -P30 -I% sh -c \
  'curl -sLk "%" -o "$OUTPUT_DIR/js_files/$(echo "%" | sed "s|https\?://||; s|[/?=&:#]|_|g")"'
print_success "JS file download complete."

# 5️⃣ & 6️⃣ Scan JS Files for Secrets
print_header "STEP 6: SCANNING JAVASCRIPT FILES FOR SECRETS"
if [ ! -f "$SECRETFINDER_PATH" ]; then
  print_info "SecretFinder not found at $SECRETFINDER_PATH. Attempting to clone..."
  git clone https://github.com/m4ll0k/SecretFinder.git
  if [ ! -f "$SECRETFINDER_PATH" ]; then
    print_error "Failed to clone SecretFinder. Please install it manually."
  fi
  print_info "Installing SecretFinder requirements..."
  python3 -m venv venv
  source venv/bin/activate
  pip install -r SecretFinder/requirements.txt
  deactivate
fi

print_info "Running SecretFinder on all downloaded JS files..."
touch "$OUTPUT_DIR/findings/secretfinder_results.txt"
for f in "$OUTPUT_DIR"/js_files/*; do
  if [ -f "$f" ]; then
    echo -e "\n\n${YELLOW}[+]> scanning $f${RESET}" >> "$OUTPUT_DIR/findings/secretfinder_results.txt"
    python3 "$SECRETFINDER_PATH" -i "$f" -o cli 2>/dev/null >> "$OUTPUT_DIR/findings/secretfinder_results.txt"
  fi
done
print_success "Secret scanning complete."

# 8️⃣ High-Confidence Secret Filtering
print_header "STEP 7: FILTERING FOR HIGH-CONFIDENCE SECRETS"
grep -iE "api[_-]?key|access[_-]?token|secret|bearer|jwt|client[_-]?id|private_key" \
  "$OUTPUT_DIR/findings/secretfinder_results.txt" > "$OUTPUT_DIR/findings/HIGH_CONFIDENCE_SECRETS.txt"
print_success "High-confidence results saved to $OUTPUT_DIR/findings/HIGH_CONFIDENCE_SECRETS.txt"

# --- FINAL SUMMARY ---
print_header "🎉 RECON COMPLETE 🎉"
print_info "All findings are saved in the '$OUTPUT_DIR' directory."
print_info "High-confidence secrets (if any) are in:"
echo -e "  ${BOLD}$OUTPUT_DIR/findings/HIGH_CONFIDENCE_SECRETS.txt${RESET}"
print_info "Next steps:"
echo "  1. Manually review $OUTPUT_DIR/findings/HIGH_CONFIDENCE_SECRETS.txt"
echo "  2. Manually inspect files in $OUTPUT_DIR/js_files/ for hidden endpoints and logic."
echo "  3. Use js-beautify on complex files: js-beautify <file> > clean.js"
