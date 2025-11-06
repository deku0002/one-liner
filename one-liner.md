subfinder -d verily.com -silent > subdomains.txt


cat subdomains.txt | httpx -silent > live-urls.txt

gospider -S live-urls.txt -c 5 -d 2 --other-source > crawl-results.txt

# This will show you all the JS, JSON, and DB files found
cat crawl-results.txt | grep -E "\.js|\.json|\.db"

## Step 5: Analyze Your Discovered Files
Now that you have a list of .js and .json files, you need to download them and search inside them for vulnerabilities.

Download the Files: (You already have the list from grep or ffuf, just wget them).

Analyze with grep: This is the final and most important step.

To find API Keys, Secrets, or Tokens:

Bash

grep -E -i "apiKey|secret|token|bearer|password|auth|key" *.js *.json
To find API Endpoints:

Bash

grep -E -i "/api/|/v1/|/v2/|/internal/|/graphql/" *.js *.json
To find Open Redirects (like you were doing):

Bash

grep -E -i "location\.href|location\.assign|location\.replace" *.js
This workflow (Subdomains -> Live Hosts -> Crawl/Fuzz -> Analyze) is a professional and effective method for finding vulnerabilities.




1st 

 subfinder -d domain.com -silent  | \
while read host; do \
  for path in /config.js /config.json /app/config.js /settings.json /database.json /firebase.json /.env /.env.production /api_keys.json /credentials.json /secrets.json /google-services.json /package.json /package-lock.json /composer.json /pom.xml /docker-compose.yml /manifest.json /service-worker.js; do \
    echo "$host$path"; \
  done; \
done | httpx -mc 200



2nd 

cat waybacks.txt | \ 
sed -E 's#(redirect=|url=|next=|return=|dest=|destination=|continue=|goto=|redirecturl=)[^&]*#\1https://evil.com#gI' | \
httpx -silent -mc 301,302,307,308 -location


3rd 

cat jsfiles.txt | xargs -I% curl -s "%" | \
grep -Eo "(http|https)://[a-zA-Z0-9./?=_-]*|/[-a-zA-Z0-9@:%_+.~#?&//=]*" | sort -u



4th

while read -r url; do curl -sL --max-time 10 "$url" 2>/dev/null | grep -Eo 'https?://[A-Za-z0-9._-]+\.firebaseio\.com|https?://[A-Za-z0-9._-]+\.firebasedatabase\.app|https?://[A-Za-z0-9._-]+\.s3\.amazonaws\.com|mongodb://[^[:space:]]+|redis://[^[:space:]]+|postgresql?://[^[:space:]]+|[Pp]assword[[:space:]]*[:=][[:space:]]*[^[:space:]]{6,128}|AIza[0-9A-Za-z_-]{35}' | sort -u | sed "s|^|$url -> |" | grep -vE '(\.env|config\.json|\.php|settings\.py|application\.yml)$'; done < jsf




5th

cat waybackurls \
  | grep -Eoi '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' \
  | tr '[:upper:]' '[:lower:]' \
  | grep -vE '\.(png|jpg|jpeg|svg|gif)$' \
  | grep -vE '^(donot.?reply|no-?reply|noreply|no_reply|webmaster|postmaster|admin|support|abuse|bounce|mailer-daemon|contact|help)@' \
  | grep -vE '^[0-9]+@' \
  | sort -u


Finding Hidden Email Leaks via Wayback : One Liner! 🔥 

Sometimes, the simplest recon tricks give you the most surprising data leaks.
While analyzing old archives for a target, I used this one-liner to extract valid email addresses buried in historic endpoints without any extra tooling.

cat waybackurls \
| grep -Eoi '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' \
| tr '[:upper:]' '[:lower:]' \
| grep -vE '\.(png|jpg|jpeg|svg|gif)$' \
| grep -vE '^(donot.?reply|no-?reply|noreply|no_reply|webmaster|postmaster|admin|support|abuse|bounce|mailer-daemon|contact|help)@' \
| grep -vE '^[0-9]+@' \
| sort -u

This helps filter out decoy emails, auto-responders, and image file references giving you a clean list of potentially sensitive addresses for investigation or correlation.
“Information leaks are often hidden in plain sight — just buried in time.”
  
