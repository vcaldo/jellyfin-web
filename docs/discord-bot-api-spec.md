# Discord Bot API Specification

API contract between Jellyfin Web ("Play on Discord" button) and the companion Discord bot.

## Configuration

The bot URL is set at build time via the `DISCORD_BOT_URL` environment variable.

| Variable | Default | Description |
|---|---|---|
| `DISCORD_BOT_URL` | `http://discord-bot:3000` | Base URL of the Discord bot API. Since both services run in the same Docker stack, this uses the Docker service name. |

**Docker Compose example:**

```yaml
services:
  jellyfin:
    # ...
    environment:
      - DISCORD_BOT_URL=http://discord-bot:3000

  discord-bot:
    image: your-discord-bot:latest
    ports:
      - "3000:3000"
```

---

## Endpoints

### `POST /api/play`

Start playback of a Jellyfin media item in a Discord voice channel.

**Headers:**

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |

**No authentication required** (internal Docker network).

#### Request Body

```jsonc
{
  // Pre-built stream URLs ready for ffmpeg consumption
  "streamUrls": {
    "directStream": "string",  // Direct stream URL (no transcoding, no subtitles)
    "transcoded": "string"     // Transcoded URL (h264+aac, with subtitle burn-in if selected)
  },

  // Full item metadata from Jellyfin
  "item": {
    "id": "string",                    // Jellyfin item ID (GUID)
    "name": "string",                  // Title (e.g. "The Matrix")
    "type": "string",                  // "Movie" | "Episode" | "Video" | "MusicVideo"
    "mediaType": "string",             // "Video"
    "container": "string",             // File container (e.g. "mkv", "mp4")
    "productionYear": "number | null", // Release year
    "overview": "string | null",       // Plot summary
    "seriesName": "string | null",     // TV show name (episodes only)
    "seasonName": "string | null",     // Season name (episodes only)
    "indexNumber": "number | null",    // Episode number
    "parentIndexNumber": "number | null", // Season number
    "runTimeTicks": "number | null",   // Duration in ticks (1 tick = 100ns, divide by 10_000_000 for seconds)
    "officialRating": "string | null", // Content rating (e.g. "PG-13")
    "communityRating": "number | null",// User rating (0-10)
    "genres": ["string"],              // Genre list
    "studios": ["string"],             // Studio names
    "imageTag": "string | null",       // Primary image tag (for building poster URL)
    "backdropImageTag": "string | null", // Backdrop image tag
    "serverId": "string"               // Jellyfin server instance ID
  },

  // Selected media source details
  "mediaSource": {
    "id": "string",            // Media source ID
    "container": "string",     // Original container format
    "bitrate": "number | null",// Total bitrate (bps)
    "size": "number | null",   // File size in bytes
    "videoStream": {
      "codec": "string",      // e.g. "h264", "hevc"
      "width": "number",      // e.g. 1920
      "height": "number",     // e.g. 1080
      "bitRate": "number"     // Video bitrate (bps)
    } | null,
    "audioStream": {
      "codec": "string",      // e.g. "aac", "ac3", "dts"
      "channels": "number",   // e.g. 2, 6, 8
      "language": "string",   // e.g. "eng", "jpn"
      "bitRate": "number"     // Audio bitrate (bps)
    } | null,
    "subtitleTracks": [        // All available subtitle tracks
      {
        "index": "number",     // Stream index for URL construction
        "language": "string | null",
        "displayTitle": "string | null",
        "codec": "string | null",  // e.g. "srt", "ass", "subrip"
        "isExternal": "boolean"
      }
    ]
  } | null,

  // Currently selected subtitle (null if none)
  "subtitle": {
    "streamIndex": "number",   // Subtitle stream index used in transcoded URL
    "method": "Encode",        // Always "Encode" (burn-in) for Discord compatibility
    "track": {                 // Matched track info from subtitleTracks
      "index": "number",
      "language": "string | null",
      "displayTitle": "string | null",
      "codec": "string | null",
      "isExternal": "boolean"
    } | null
  } | null,

  // Jellyfin server base URL
  "serverUrl": "string"       // e.g. "http://jellyfin:8096"
}
```

#### Example Request

```json
{
  "streamUrls": {
    "directStream": "http://jellyfin:8096/Videos/abc123/stream.mkv?Static=true&mediaSourceId=abc123&deviceId=discord-bot&ApiKey=token123",
    "transcoded": "http://jellyfin:8096/Videos/abc123/stream?mediaSourceId=abc123&deviceId=discord-bot&ApiKey=token123&VideoCodec=h264&AudioCodec=aac&MaxStreamingBitrate=8000000&SubtitleStreamIndex=2&SubtitleMethod=Encode"
  },
  "item": {
    "id": "abc123",
    "name": "The Matrix",
    "type": "Movie",
    "mediaType": "Video",
    "container": "mkv",
    "productionYear": 1999,
    "overview": "A computer hacker learns about the true nature of reality...",
    "seriesName": null,
    "seasonName": null,
    "indexNumber": null,
    "parentIndexNumber": null,
    "runTimeTicks": 81360000000,
    "officialRating": "R",
    "communityRating": 8.7,
    "genres": ["Action", "Sci-Fi"],
    "studios": ["Warner Bros."],
    "imageTag": "abc123tag",
    "backdropImageTag": "backdrop456",
    "serverId": "server789"
  },
  "mediaSource": {
    "id": "abc123",
    "container": "mkv",
    "bitrate": 10000000,
    "size": 8500000000,
    "videoStream": {
      "codec": "h264",
      "width": 1920,
      "height": 1080,
      "bitRate": 8000000
    },
    "audioStream": {
      "codec": "aac",
      "channels": 6,
      "language": "eng",
      "bitRate": 384000
    },
    "subtitleTracks": [
      {
        "index": 2,
        "language": "eng",
        "displayTitle": "English",
        "codec": "srt",
        "isExternal": false
      },
      {
        "index": 3,
        "language": "por",
        "displayTitle": "Portuguese (Brazil)",
        "codec": "subrip",
        "isExternal": false
      }
    ]
  },
  "subtitle": {
    "streamIndex": 2,
    "method": "Encode",
    "track": {
      "index": 2,
      "language": "eng",
      "displayTitle": "English",
      "codec": "srt",
      "isExternal": false
    }
  },
  "serverUrl": "http://jellyfin:8096"
}
```

#### Response

**Success (200):**

```json
{
  "status": "ok",
  "message": "Playback started"
}
```

**Error (4xx/5xx):**

```json
{
  "status": "error",
  "message": "Description of what went wrong"
}
```

---

## Stream URL Formats

The `streamUrls` object contains two pre-built URLs. The bot should pick the appropriate one:

### `directStream` — No subtitles, direct file streaming

```
{serverUrl}/Videos/{itemId}/stream.{container}?Static=true&mediaSourceId={id}&deviceId=discord-bot&ApiKey={token}
```

- Preferred when no subtitles are needed
- Passes the original file through without re-encoding
- Lowest latency and highest quality

### `transcoded` — With subtitle burn-in

```
{serverUrl}/Videos/{itemId}/stream?mediaSourceId={id}&deviceId=discord-bot&ApiKey={token}&VideoCodec=h264&AudioCodec=aac&MaxStreamingBitrate=8000000[&SubtitleStreamIndex={idx}&SubtitleMethod=Encode]
```

- Forces server-side transcoding to h264+aac
- Subtitles are burned into the video stream (only viable method for Discord)
- Capped at 8 Mbps
- If no subtitles were selected, this URL still forces transcode but without subtitle parameters

### Bot decision logic

```
if payload.subtitle is not null:
    use streamUrls.transcoded   # subtitles need burn-in
else:
    use streamUrls.directStream # no subs, prefer direct
```

---

## Building Image URLs

The bot can construct Jellyfin image URLs from the payload if needed (e.g. for Discord embeds):

```
{serverUrl}/Items/{item.id}/Images/Primary?tag={item.imageTag}
{serverUrl}/Items/{item.id}/Images/Backdrop?tag={item.backdropImageTag}
```

---

## What Was Implemented (Jellyfin Web Side)

### Files Modified

| File | Change |
|---|---|
| `src/controllers/itemDetails/index.html` | Added "Play on Discord" button with `headphones` icon |
| `src/controllers/itemDetails/index.js` | Added click handler, event binding, visibility logic, and `POST /api/play` call |
| `webpack.common.js` | Added `__DISCORD_BOT_URL__` compile-time constant from `DISCORD_BOT_URL` env var |

### Button Behavior

1. **Visibility**: The button appears only for playable video items (`Movie`, `Episode`, `Video`, `MusicVideo`)
2. **On click**:
   - Reads the currently selected media source, audio track, and subtitle track from the UI dropdowns
   - Constructs both stream URLs (direct + transcoded with burn-in)
   - Gathers all available item metadata, media source details, and subtitle track info
   - Sends the full payload to `POST {DISCORD_BOT_URL}/api/play`
   - Logs the payload to the browser console with `[Play on Discord]` prefix
   - Shows a toast notification on success or failure

### What the Bot Needs to Implement

The companion Discord bot must expose `POST /api/play` on port 3000 (or whichever port `DISCORD_BOT_URL` points to). It receives the payload above and should:

1. Parse `streamUrls` and `subtitle` to decide which URL to use
2. Use `item` metadata for Discord embed messages (title, year, poster image, etc.)
3. Feed the chosen stream URL to ffmpeg for voice channel playback
