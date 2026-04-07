# Building a Listener App

A practical guide for developers building a MixMates Listener app. Read the [API reference](https://jbaddnz.github.io/mixmates-listener-api/) first, then come back here for implementation guidance.

## Getting a Token

Before your app can call the API, the user needs a Bearer token:

1. Sign up at [mixmat.es](https://mixmat.es) via Spotify, Tidal, or Apple Music
2. Upgrade to a paid plan ($5/mo or above) — audio recognition is a paid feature
3. Open **Settings** → scroll to **Listening** → toggle **Listen** on
4. Open the **Listen** group (via the group selector) → click **Listen settings**
5. Click **Listen Key** → copy the token

Your app should provide a way to paste or scan this token. Store it securely on the device (Keychain on iOS, EncryptedSharedPreferences on Android).

The token doesn't expire unless the user explicitly revokes it. Your app should handle `401` responses gracefully — prompt the user to generate a new token if theirs has been revoked.

## Audio Recording

### What works best

- **Duration:** 8–15 seconds. 11 seconds is the sweet spot — long enough for reliable recognition, short enough to be quick
- **Format:** Any common audio format works. The recognition engine accepts whatever you send. Tested formats: `audio/webm` (Opus), `audio/mp4` (AAC), `audio/wav`
- **Sample rate:** 44.1kHz or 48kHz. Lower rates (16kHz, 22kHz) still work but reduce accuracy
- **Quality:** 64–128kbps is plenty. Higher bitrate doesn't improve recognition — it just makes the upload slower
- **File size:** Max 5MB. An 11-second webm/opus recording at 128kbps is typically 150–200KB

### Platform-specific notes

**Android (MediaRecorder):**
```
MediaRecorder.AudioEncoder.AAC
MediaRecorder.OutputFormat.MPEG_4
sampleRate: 44100
bitRate: 128000
```

**iOS (AVAudioRecorder):**
```
AVFormatIDKey: kAudioFormatMPEG4AAC
AVSampleRateKey: 44100
AVNumberOfChannelsKey: 1
AVEncoderBitRateKey: 128000
```

**Web (MediaRecorder API):**
```javascript
new MediaRecorder(stream, { mimeType: 'audio/webm;codecs=opus' })
```

### Tips

- **Mono is fine.** Song recognition doesn't benefit from stereo
- **Don't apply noise reduction or filters.** Send the raw audio — the recognition engine handles noisy environments better than you might expect
- **Start recording immediately.** Don't wait for silence detection or audio level thresholds — just capture what's playing

## Core Flow

```
┌──────────────┐
│  User taps   │
│  "Listen"    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Record     │
│  11 seconds  │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌─────────────┐
│ POST         │────▶│  "saved"    │──▶ Show track + platform links
│ /recognize   │     │  "duplicate"│──▶ Show track (already in queue)
│              │     │  "no_match" │──▶ "Couldn't identify that"
│              │     │  "no_links" │──▶ "Identified but not on streaming"
└──────────────┘     └─────────────┘
       │
       ▼ (optional)
┌──────────────┐
│  User taps   │
│   "Share"    │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌─────────────┐
│ GET /groups  │────▶│ Show group  │
│              │     │   picker    │
└──────────────┘     └──────┬──────┘
                            │
                            ▼
                     ┌─────────────┐
                     │    POST     │
                     │ /history/   │
                     │  {id}/share │
                     └─────────────┘
```

## Handling Each Outcome

### `saved`

The track was recognised and saved to the user's listen queue. Show:
- Track title and artist
- Thumbnail image (from `track.thumbnail`)
- Platform links (tap to open in Spotify/Tidal/Apple Music)
- Share button (to share to groups)
- A link or button to open the track in MixMates (`track.share_url`)

### `duplicate`

The track was recognised but it's already in the listen queue. Same display as `saved` — the user probably wants to see the track info regardless. You might show a subtle "already captured" indicator.

### `no_match`

The recognition engine couldn't identify the audio. `track` is `null`. Show a friendly message:
- "Couldn't catch that one"
- "Try again — hold your phone closer to the speaker"
- Offer to retry

Don't be too apologetic. Recognition works well in most environments but isn't perfect. Loud background noise, live performances of obscure tracks, and DJ mixes with heavy effects are common failure cases.

### `no_links`

The track was identified (title and artist are available in `track`) but no streaming platform URLs were found. This is rare — it usually means the track exists on niche platforms but not on Spotify, Tidal, or Apple Music. Show the title and artist, and explain that streaming links aren't available.

### `error` (502)

The recognition service is temporarily unavailable. Show an error and let the user retry. Don't retry automatically — if the service is down, hammering it won't help.

## Rate Limits

Check the `X-RateLimit-Remaining` header after each `/recognize` call. When it hits 0, show the user how long until they can try again (from `X-RateLimit-Reset`).

Limits:
- **Paid users:** 20 recognitions per hour
- **VIP users:** 100 per hour
- **Global:** 200 per day across all users

Your app should:
1. Show remaining recognitions somewhere accessible (not in-your-face, but findable)
2. Disable the record button when remaining is 0
3. Show a countdown to the reset time

Don't hide rate limits from users — they're paying for the service and deserve to know their quota.

## Offline and Connectivity

The API has no batch or offline mode. Each recognition requires an active internet connection.

If your app wants to support offline recording:
1. Record audio to local storage
2. Show a queue of pending recordings
3. Submit them one at a time when connectivity returns
4. Handle each result individually (some may succeed, others may not)

Keep queued recordings for a reasonable time (24-48 hours). Audio recognition accuracy doesn't degrade with a short delay — the audio clip is what matters, not when it was captured.

## Opening Platform Links

When showing results, let users tap platform links to open the track directly in their streaming app:

- **Spotify:** `https://open.spotify.com/track/...` — opens in Spotify app if installed
- **Tidal:** `https://tidal.com/browse/track/...` — opens in Tidal app if installed
- **Apple Music:** `https://music.apple.com/...` — opens in Apple Music

On Android, these URLs trigger app deep links automatically. On iOS, the same applies for installed apps.

The `share_url` (`https://mixmat.es/{shortcode}`) opens the MixMates share page, which shows all platforms. Useful as a fallback or for sharing outside the app.

## Token Management

- Store the token securely (OS keychain, not plain text)
- Handle `401` gracefully — token may have been revoked
- Call `GET /auth/me` on app launch to verify the token is still valid and check quota
- Don't hardcode tokens in source code or include them in logs

## Testing

Use `GET /health` to verify connectivity without consuming a recognition. Returns `200` with `{ data: { status: "ok" } }` regardless of auth state.

Use `GET /auth/me` to verify your token works and check the user's current rate limit status before recording.
