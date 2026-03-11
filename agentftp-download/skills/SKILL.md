---
name: agentftp-download
description: Download files from AgentFTP by URL or artifact ID
version: 1.0.0
---

# AgentFTP Download Skill

Download artifacts from AgentFTP using a public URL or artifact ID.
No authentication required for downloads.

## Usage

When the user provides an AgentFTP URL or artifact ID to download:

1. Extract the artifact ID from the URL (last path segment)
2. Get metadata to learn the filename
3. Download using the presigned URL

## Download Flow

```bash
# Extract artifact ID from URL like https://agentftp.com/a/ABC123
ARTIFACT_ID="ABC123"

# Get artifact metadata
META=$(curl -s https://api.agentftp.com/artifacts/$ARTIFACT_ID)
FILENAME=$(echo "$META" | python3 -c "import sys,json; print(json.load(sys.stdin)['filename'])")

# Get download URL
DL_RESPONSE=$(curl -s -H "Accept: application/json" \
  https://api.agentftp.com/artifacts/$ARTIFACT_ID/download)
DOWNLOAD_URL=$(echo "$DL_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['download_url'])")

# Download the file
curl -s -o "$FILENAME" "$DOWNLOAD_URL"

echo "Downloaded: $FILENAME"

# If it's a zip, extract it
if [[ "$FILENAME" == *.zip ]]; then
  EXTRACT_DIR="${FILENAME%.zip}"
  mkdir -p "$EXTRACT_DIR"
  unzip -o "$FILENAME" -d "$EXTRACT_DIR"
  rm "$FILENAME"
  echo "Extracted to: $EXTRACT_DIR/"
fi
```

## URL Formats

The skill handles these URL patterns:
- `https://agentftp.com/a/ARTIFACT_ID`
- `https://api.agentftp.com/artifacts/ARTIFACT_ID`
- Just the artifact ID: `ARTIFACT_ID`
