<img src="../../assets/mml-text.svg" alt="MixMat Ltd" width="120">

# MixMates Listener API v1

Base URL: `https://mixmat.es/api/v1/listener`

## Authentication

All endpoints require a Bearer token (except `/health`):

```
Authorization: Bearer YOUR_TOKEN
```

Generate a token in MixMates Settings or via `POST /auth/token`.
Requires a paid MixMates account with Listen enabled.

## Quick Start

```bash
# Check the API is up
curl https://mixmat.es/api/v1/listener/health

# Verify your token
curl -H "Authorization: Bearer TOKEN" \
  https://mixmat.es/api/v1/listener/auth/me

# Recognise a song
curl -X POST \
  -H "Authorization: Bearer TOKEN" \
  -F "audio=@recording.webm" \
  https://mixmat.es/api/v1/listener/recognize

# List your listen queue
curl -H "Authorization: Bearer TOKEN" \
  https://mixmat.es/api/v1/listener/history

# Share a track to a group
curl -X POST \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"group_ids": ["GROUP_ID"]}' \
  https://mixmat.es/api/v1/listener/history/ITEM_ID/share
```

## Response Format

**Success:**

```json
{
  "data": { ... },
  "meta": {
    "request_id": "req_a1b2c3d4e5f6",
    "timestamp": "2026-04-07T10:00:00.000Z"
  }
}
```

**Error:**

```json
{
  "error": {
    "code": "auth_required",
    "message": "Authentication required"
  },
  "meta": { ... }
}
```

## Rate Limits

Every response includes:

```
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 17
X-RateLimit-Reset: 1709654400
```

Recognition limits: 20/hr (paid), 100/hr (VIP), 200/day (global).
`429` responses include `Retry-After`.

## Endpoints

### `GET /health`

No auth. Returns `{ status: "ok", version: "1" }`.

---

### `POST /recognize`

Submit audio for recognition.

**Request:** `multipart/form-data` with `audio` field (max 5MB).

**Response:**

```json
{
  "data": {
    "status": "saved",
    "source": "recognition",
    "track": {
      "title": "Midnight City",
      "artist": "M83",
      "thumbnail": "https://...",
      "shortcode": "aBcDeF12",
      "share_url": "https://mixmat.es/aBcDeF12",
      "platforms": {
        "spotify": "https://open.spotify.com/track/...",
        "tidal": "https://tidal.com/browse/track/...",
        "appleMusic": "https://music.apple.com/..."
      }
    }
  }
}
```

**Status values:**

| Status | Meaning |
|---|---|
| `saved` | Track identified and saved to listen queue |
| `duplicate` | Track already in listen queue |
| `no_match` | Could not identify the audio |
| `no_links` | Identified but no streaming URLs found |

`track` is `null` when status is `no_match`.

---

### `GET /history`

Paginated listen queue.

**Parameters:** `cursor` (string), `limit` (1-50, default 20)

**Response:**

```json
{
  "data": {
    "items": [
      {
        "id": "abc123",
        "title": "Midnight City",
        "artist": "M83",
        "thumbnail": "https://...",
        "shortcode": "aBcDeF12",
        "share_url": "https://mixmat.es/aBcDeF12",
        "platforms": { ... },
        "created_at": "2026-04-07T10:00:00.000Z"
      }
    ],
    "cursor": "1709654400000",
    "has_more": true
  }
}
```

---

### `GET /history/:id`

Single history entry with share info.

**Response:** Same as history item plus `shared_to` array:

```json
{
  "data": {
    "id": "abc123",
    "title": "...",
    "shared_to": [
      { "group_id": "g1", "group_name": "Wellington Batucada" }
    ],
    ...
  }
}
```

---

### `DELETE /history/:id`

Remove an entry from your listen queue.

---

### `POST /history/:id/share`

Share to groups.

**Body:**

```json
{
  "group_ids": ["g1", "g2"]
}
```

Max 20 groups per request. You must be a member of each group.

---

### `GET /groups`

Your groups (for share target selection). Excludes the Listen group.

```json
{
  "data": {
    "items": [
      { "id": "g1", "name": "Wellington Batucada", "description": "..." }
    ]
  }
}
```

---

### `GET /recordings`

List saved audio recordings (max 5, expire after 30 days).

```json
{
  "data": {
    "items": [
      {
        "recording_id": "1709654400:a1b2c3",
        "created_at": "2026-04-07T10:00:00.000Z",
        "outcome": "matched",
        "title": "Midnight City",
        "artist": "M83",
        "mime_type": "audio/webm"
      }
    ]
  }
}
```

---

### `GET /recordings/:recording_id`

Download audio blob. Returns binary with correct `Content-Type`. Not wrapped in JSON.

---

### `DELETE /recordings`

Delete all saved recordings.

---

### `POST /auth/token`

Generate a new Bearer token. **Session auth only** (from the web UI).

```json
{
  "data": {
    "token": "...",
    "created_at": "2026-04-07T10:00:00.000Z"
  }
}
```

---

### `DELETE /auth/token`

Revoke your current token. All tokens issued before this point become invalid. **Session auth only.**

---

### `GET /auth/me`

Verify your token and check quota. Accepts both Bearer and session auth.

```json
{
  "data": {
    "user": {
      "id": "abc123",
      "display_name": "Jamie",
      "role": "paid",
      "listen_enabled": true,
      "preferred_platform": "tidal"
    },
    "rate_limit": {
      "limit": 20,
      "remaining": 17,
      "reset_at": 1709654400
    }
  }
}
```

## Error Codes

| Code | HTTP | Meaning |
|---|---|---|
| `auth_required` | 401 | No token or token invalid |
| `auth_invalid_token` | 401 | Token could not be decrypted |
| `auth_insufficient_role` | 403 | Paid account required |
| `auth_listen_disabled` | 403 | Listen not enabled in Settings |
| `auth_token_revoked` | 401 | Token was revoked |
| `rate_limit_user` | 429 | Per-user hourly limit exceeded |
| `rate_limit_global` | 429 | Daily global limit exceeded |
| `invalid_content_type` | 400 | Request not multipart/form-data |
| `missing_audio` | 400 | No audio file in request |
| `audio_too_large` | 400 | Audio exceeds 5MB |
| `invalid_body` | 400 | Invalid JSON body |
| `missing_field` | 400 | Required field missing |
| `invalid_field` | 400 | Field value invalid |
| `not_found` | 404 | Resource not found |
| `recognition_unavailable` | 502 | Recognition service down |
