# upload-media-transcripts

A Python script for uploading caption/transcript files to PeerTube videos, driven by a Google Sheet that maps video metadata to caption files.

---

## Overview

This script reads a Google Sheet containing video records, looks up each video on a PeerTube instance by its short UUID, and uploads an associated caption file (`.srt` or `.vtt`) via the PeerTube API. It supports dry-run mode, flexible configuration via a YAML file, and command-line overrides for most settings.

---

## Requirements

**Python 3.10+** (uses `dict | None` union syntax)

Install dependencies:

```bash
pip install \
  pyyaml \
  pandas \
  requests \
  urllib3 \
  langcodes \
  google-auth \
  google-api-python-client \
  openpyxl
```

---

## Configuration

The script requires a YAML configuration file. Below is the expected structure:

```yaml
log_file: /path/to/logfile.log
google_credentials_file: /path/to/google-service-account.json
google_sheet_id: "your_google_sheet_id"
google_sheet_name: "Sheet1"

environments:
  production:
    peertube_instance: https://your-peertube-instance.example.com
    peertube_username: admin
    peertube_password: yourpassword
    peertube_channel: 1
  staging:
    peertube_instance: https://staging.example.com
    peertube_username: admin
    peertube_password: stagingpassword
    peertube_channel: 2
```

Multiple environments can be defined under `environments:`. The `--env` argument selects which one to use at runtime.

---

## Google Sheet Format

The script reads from a Google Sheet with the following columns:

| Column | Required | Description |
|--------|----------|-------------|
| `id` | Yes | A local identifier for the video record |
| `title` | Yes | Title of the video |
| `file` | Yes | Path to the local video/media file |
| `field_media_oembed_video` | Yes | Full URL to the PeerTube video (used to extract the short UUID) |
| `transcript` | No | Path to the caption file (`.srt` or `.vtt`) |
| `transcript_language` | No | Language name or code for the caption (e.g. `English`, `en`, `fra`) |

If `transcript_language` is omitted or blank, it defaults to `zxx` (no linguistic content).

---

## Usage

```bash
python upload-media-transcripts.py \
  --config-file config.yaml \
  --log-file run.log \
  --env production
```

### All Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--config-file` | Yes | Path to the YAML configuration file |
| `--log-file` | Yes | Path to the log output file |
| `--env` | Yes | Environment key from the config file (e.g. `production`) |
| `--dry-run` | No | `True` or `False` (default: `False`). Skips actual API uploads |
| `--in-google-sheet-id` | No | Override the Google Sheet ID from config |
| `--in-google-sheet-name` | No | Override the sheet tab name from config |
| `--in-google-creds-file` | No | Override the Google credentials file from config |
| `--peertube-instance` | No | Override the PeerTube instance URL |
| `--peertube-username` | No | Override the PeerTube username |
| `--peertube-password` | No | Override the PeerTube password |
| `--peertube-channel` | No | Override the PeerTube channel number |

### Dry Run Example

Validates configuration and logs what would be uploaded without making any API calls:

```bash
python upload-media-transcripts.py \
  --config-file config.yaml \
  --log-file dry-run.log \
  --env staging \
  --dry-run True
```

---

## How It Works

1. **Login** — Authenticates with the PeerTube instance using OAuth2 and retrieves a bearer token.
2. **Read Sheet** — Reads the Google Sheet into a DataFrame using a service account.
3. **Process Rows** — For each row:
   - Extracts the PeerTube short UUID from the `field_media_oembed_video` URL.
   - Resolves the transcript language to a PeerTube-recognized language code via the `/api/v1/videos/languages` endpoint.
   - Uploads the caption file using a `PUT` request to `/api/v1/videos/{uuid}/captions/{language}`.
4. **Log Results** — Each upload result (success or error) is logged to the log file and printed to stdout.

### Caption Upload Responses

| Status | Meaning |
|--------|---------|
| `204` | Caption uploaded successfully (expected success response) |
| `200` | Caption accepted with a JSON body (non-standard success) |
| `400` | Bad request — likely invalid language code or file format |
| `404` | Video UUID or language not found on the instance |

---

## Google Service Account Setup

The script authenticates with Google Sheets using a service account JSON key file.

1. Create a service account in the [Google Cloud Console](https://console.cloud.google.com/).
2. Grant it the **Editor** role on the target Google Sheet (share the sheet with the service account's email).
3. Download the JSON key file and reference it in your config or via `--in-google-creds-file`.

---

## Logging

All activity is written to the log file specified by `--log-file`. The log format is:

```
YYYYMMDD HH:MM:SS.mmm LEVEL message
```

Errors are also printed to stdout for visibility during interactive runs.

---

## Notes

- SSL verification is disabled for PeerTube requests (`verify=False`). This suppresses certificate warnings for self-hosted instances with self-signed certificates. For production deployments with valid certificates, you may want to re-enable verification.
- Language matching supports both ISO codes (e.g. `en`, `fra`) and full language names (e.g. `English`, `French`), resolved against the live PeerTube languages API. The result is cached for the duration of the run.
- The script calls `exit()` on unrecoverable errors (missing required columns, failed login) but logs and continues on per-row caption upload failures.
