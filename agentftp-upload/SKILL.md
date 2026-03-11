---
name: agentftp-upload
description: Upload files or folders to AgentFTP and get a shareable URL
version: 1.0.0
---

# AgentFTP Upload Skill

Upload files or folders to AgentFTP to share with other agents or humans.
Returns a public URL that anyone can use to download the artifact.

## Setup

This skill requires authentication. If not yet authenticated, it will
guide you through the GitHub sign-in process.

Config file location: `~/.agentftp/config.json`

## Usage

When the user asks to upload, share, or publish a file or folder:

1. First check if authenticated by reading `~/.agentftp/config.json`
2. If no config exists, run the auth flow (see below)
3. Zip the folder if needed, then upload via the API

## Auth Flow (if not authenticated)

```bash
# Start auth session
AUTH_RESPONSE=$(curl -s https://api.agentftp.com/auth/login)
SESSION_ID=$(echo "$AUTH_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['session_id'])")
AUTH_URL=$(echo "$AUTH_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['auth_url'])")

# Tell the user to open the URL
echo "Please open this URL to authenticate: $AUTH_URL"

# Then poll for the token (the user needs to complete auth in browser first)
# Poll every 2 seconds until complete:
curl -s "https://api.agentftp.com/auth/token?session=$SESSION_ID"
```

After receiving the token, save it:
```bash
mkdir -p ~/.agentftp
cat > ~/.agentftp/config.json << 'CONF'
{"token": "THE_TOKEN", "username": "THE_USERNAME", "api_base": "https://api.agentftp.com"}
CONF
chmod 600 ~/.agentftp/config.json
```

## Upload Flow

```bash
# Read the saved token
TOKEN=$(python3 -c "import json; print(json.load(open('$HOME/.agentftp/config.json'))['token'])")

# If uploading a directory, zip it first
cd /path/to/parent && zip -r /tmp/artifact.zip folder_name/

# Get file size
SIZE=$(stat -f%z /tmp/artifact.zip 2>/dev/null || stat -c%s /tmp/artifact.zip)

# Create artifact and get presigned upload URL
RESPONSE=$(curl -s -X POST https://api.agentftp.com/artifacts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"filename\": \"artifact.zip\", \"size\": $SIZE, \"description\": \"Uploaded by agent\"}")

UPLOAD_URL=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['upload_url'])")
PUBLIC_URL=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['public_url'])")

# Upload the file
curl -s -X PUT "$UPLOAD_URL" \
  -H "Content-Type: application/zip" \
  --data-binary @/tmp/artifact.zip

# Clean up
rm /tmp/artifact.zip

echo "Uploaded! Public URL: $PUBLIC_URL"
```

## Limits

- Max file size: 100MB
- Max artifacts per user: 50
- Artifacts expire after 7 days
