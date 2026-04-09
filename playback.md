# DVR Playback Feature — Technical Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Endpoints Summary](#endpoints-summary)
4. [Detailed Endpoint Reference](#detailed-endpoint-reference)
   - [`getDvrPlayback`](#1-getdvrplayback)
   - [`dvr_proxy`](#2-dvr_proxy)
5. [Remote Edge Call — `playbackurl`](#remote-edge-call--playbackurl)
6. [End-to-End Playback Flow](#end-to-end-playback-flow)
7. [M3U8 Proxy Rewriting](#m3u8-proxy-rewriting)
8. [Authentication & Security](#authentication--security)
9. [Error Handling](#error-handling)

---

## Overview

The **DVR Playback** feature allows authenticated users to retrieve and stream recorded video footage from Hikvision NVR/DVR devices. It is built around two key endpoints:

| Endpoint | Purpose |
|---|---|
| `getDvrPlayback` | Fetches the site/camera hierarchy **or** proxies playback/scan requests to an edge device |
| `dvr_proxy` | Streams upstream video content (`.m3u8` playlists and `.ts` segments) back to the client |

The architecture uses a **server-side proxy pattern** to solve two critical problems:

1. **CORS / Mixed-Content** — NVRs are on private networks; the browser/app cannot reach them directly.
2. **Credential Isolation** — DVR usernames and passwords never leave the backend.

---

## Architecture Diagram

```mermaid
sequenceDiagram
    participant Client as Client (Flutter)
    participant Django as Django Backend
    participant Edge as Edge Device
    participant NVR as NVR / DVR

    Client->>Django: GET /playback/get/?device_id=...&camera_name=...&date=...&start_time=...
    Django->>Edge: GET {edge_base_url}/get_playback_url/?dvr_host=...&camera_name=...&date=...&start_time=...
    Edge->>NVR: Generate RTSP URL via HikvisionURLGenerator
    NVR-->>Edge: RTSP/HLS URL
    Edge-->>Django: JSON { "rtsp_url": "http://..." }
    Django->>Django: Rewrite URL → /dvr/proxy/?url=<encoded>
    Django-->>Client: JSON { "url": "/api/livestream/dvr/proxy/?url=..." }
    Client->>Django: GET /dvr/proxy/?url=<encoded_m3u8>
    Django->>Edge: Fetch upstream .m3u8
    Django->>Django: Rewrite all segment/key URIs in playlist
    Django-->>Client: Rewritten .m3u8 playlist
    Client->>Django: GET /dvr/proxy/?url=<segment.ts>
    Django->>Edge: Stream segment bytes
    Django-->>Client: StreamingHttpResponse (video chunk)
```

---

## Endpoints Summary

| Route | View | Method | Auth | Purpose |
|---|---|---|---|---|
| `/api/livestream/playback/get/` | `getDvrPlayback` | GET | JWT / Session | Fetch playback hierarchy **or** proxy a playback/scan request to an edge device |
| `/api/livestream/dvr/proxy/` | `dvr_proxy` | GET | JWT / Session | Stream-proxy any upstream URL (`.m3u8` rewriting included) |

---

## Detailed Endpoint Reference

### 1. `getDvrPlayback`

> **URL**: `/api/livestream/playback/get/`  
> **Name**: `flutter_get_dvr_playback`  
> **Method**: `GET`  
> **Auth**: JWT or Session (required)

This is a **dual-purpose** endpoint. Its behavior changes based on whether `device_id` is provided.

---

#### Mode A — Playback / Scan Proxy (when `device_id` is provided)

Proxies the request to the appropriate edge device to either fetch a playback URL or scan storage for available recordings.

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `device_id` | int | ✅ | Primary key of the `EdgeDevice` to route the request to |
| `action` | string | ❌ | `"playback"` (default) or `"scan"`. Controls which edge endpoint is called |
| `dvr_id` | int | ❌ | Primary key of the target `NVR`. If provided, its `ip_address` is resolved and injected as `dvr_host` / `host` |
| `dvr_host` | string | ❌ | IP address of the DVR (auto-populated if `dvr_id` is given) |
| `camera_name` | string | ❌ | Name of the camera to play back |
| `channel_id` | string | ❌ | Channel ID of the camera. For `scan` action, also mapped to `track_id` if `track_id` is absent |
| `date` | string | ❌ | Target date for playback (format: `YYYY-MM-DD`) |
| `start_time` | string | ❌ | Start time for playback (format: `HH:MM:SS`) |

**Internal Routing Logic:**

| `action` value | Edge endpoint called | HTTP method |
|---|---|---|
| `"playback"` (default) | `{edge_base_url}/get_playback_url/` | `GET` (query params) |
| `"scan"` | `{edge_base_url}/scan_storage/` | `POST` (JSON body) |

**How `dvr_id` resolution works:**

When `dvr_id` is provided, the backend looks up the `NVR` record and injects the NVR's `ip_address` into the payload as both `dvr_host` and `host`. This means the client doesn't need to know the NVR's IP — just its database ID.

```python
target_nvr = NVR.all_objects.get(pk=dvr_id)
payload["dvr_host"] = target_nvr.ip_address
payload["host"] = target_nvr.ip_address
```

**Response Transformation (for `action=playback` only):**

The JSON response from the edge device is inspected and URLs are rewritten:

| Condition | Rewrite |
|---|---|
| `item["url"]` starts with `http://` or `https://` | → `/api/livestream/dvr/proxy/?url=<url_encoded>` |
| `item["rtsp_url"]` starts with `http://`, `https://`, or `rtsp://` | → `/api/livestream/dvr/proxy/?url=<url_encoded>` |

For `action=scan`, the response is returned **as-is** (no URL rewriting).

**Example Playback Request:**
```http
GET /api/livestream/playback/get/?device_id=5&dvr_id=12&camera_name=Entrance&date=2026-04-09&start_time=14:00:00
Authorization: Bearer <jwt_token>
```

**Example Playback Response:**
```json
{
  "url": "/api/livestream/dvr/proxy/?url=http%3A%2F%2F192.168.1.100%3A8080%2Fhls%2Fplayback%2Findex.m3u8",
  "rtsp_url": "rtsp://admin:pass@192.168.1.64/Streaming/tracks/101?starttime=20260409T140000Z"
}
```

**Example Scan Request:**
```http
GET /api/livestream/playback/get/?device_id=5&dvr_id=12&channel_id=101&action=scan
Authorization: Bearer <jwt_token>
```

**Example Scan Response:**
```json
[
  { "date": "2026-04-09", "start": "08:00:00", "end": "18:00:00" },
  { "date": "2026-04-08", "start": "00:00:00", "end": "23:59:59" }
]
```

---

#### Mode B — Hierarchy (when `device_id` is NOT provided)

Returns the full site/device/camera hierarchy that the authenticated user is scoped to see. This is used by the client to populate the playback UI (site selector, camera list, etc.).

**Query Parameters:** None required.

**Response Structure:**
```json
{
  "sites": [
    {
      "id": 1,
      "name": "HQ Campus",
      "city": { "id": 3, "name": "Mumbai" },
      "edge_devices": [
        { "device_id": 5, "name": "Edge-01" }
      ]
    }
  ],
  "cameras": [
    {
      "id": 42,
      "name": "Lobby Cam",
      "channel_no": 1,
      "display_name": "Lobby Cam (CH 1)",
      "nvr__id": 12,
      "nvr__name": "NVR-Main",
      "nvr__is_active": true,
      "nvr__edge_device__id": 5,
      "nvr__edge_device__name": "Edge-01",
      "nvr__edge_device__site__id": 1,
      "nvr__edge_device__site__name": "HQ Campus"
    }
  ]
}
```

> [!NOTE]
> The hierarchy is scoped via `apply_user_scope()` — users only see sites/devices/cameras they are authorized to access through the RBAC system.

---

### 2. `dvr_proxy`

> **URL**: `/api/livestream/dvr/proxy/`  
> **Name**: `flutter_dvr_proxy`  
> **Method**: `GET`  
> **Auth**: JWT or Session (required)

A **general-purpose streaming reverse proxy** that fetches any upstream URL and streams it back to the client. Contains special handling for `.m3u8` HLS playlists to rewrite internal URLs.

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `url` | string | ✅ | The upstream URL to fetch and proxy |

**Behavior:**

```mermaid
flowchart TD
    A["Incoming request with ?url=upstream_url"] --> B{URL ends with .m3u8?}
    B -- Yes --> C["Fetch full playlist text"]
    C --> D["Rewrite all segment & key URIs\nthrough dvr_proxy"]
    D --> E["Return rewritten playlist\nContent-Type: application/vnd.apple.mpegurl"]
    B -- No --> F["Stream raw bytes in\n512KB chunks"]
    F --> G["Return StreamingHttpResponse\nwith original Content-Type"]
```

**Headers Forwarded Upstream:**

| Header | Forwarded |
|---|---|
| `User-Agent` | ✅ |
| `Referer` | ✅ |
| `Cookie` | ✅ |
| `Authorization` | ✅ |
| `Range` | ✅ |

**Response Headers Set:**

| Header | Value | Applies To |
|---|---|---|
| `Access-Control-Allow-Origin` | `*` | All responses |
| `Accept-Ranges` | `bytes` | Non-`.m3u8` responses |
| `Content-Length` | *(from upstream)* | Non-`.m3u8` responses (if upstream provides it) |
| `Cache-Control` | `no-cache, no-store, must-revalidate` | `.m3u8` responses only |

**Timeout:** `5 seconds` (connect) / `60 seconds` (read)

**SSL Verification:** Disabled (`verify=False`) — required because NVRs typically use self-signed certificates.

---

## Remote Edge Call — `playbackurl`

When `getDvrPlayback` proxies a playback request to an edge device (via `GET {edge_base_url}/get_playback_url/`), the edge device runs the `playbackurl` function. Here is how it works:

### Parameters Received

| GET Parameter | Type | Required | Description |
|---|---|---|---|
| `dvr_host` | string | ✅ | IP address of the NVR/DVR (e.g., `192.168.1.64`) |
| `camera_name` | string | ✅ | Name of the camera as stored in the `Camera_model` table |
| `date` | string | ✅ | Target date (format: `YYYY-MM-DD`) |
| `start_time` | string | ✅ | Start time for playback (format: `HH:MM:SS`) |

### Processing Steps

```mermaid
flowchart TD
    A["Receive request with\ndvr_host, camera_name, date, start_time"] --> B["Lookup Camera_model:\nfilter by dvr.host AND name"]
    B --> C{Camera found?}
    C -- Yes --> D["Extract main_stream_id\ne.g. 101"]
    C -- No --> D2["main_stream_id = None"]
    D --> E["Lookup DvrInfo by host"]
    D2 --> E
    E --> F{DvrInfo exists?}
    F -- No --> G["Return 404:\n{ error: DVR not found }"]
    F -- Yes --> H["Create HikvisionURLGenerator\nwith DVR host, username, password"]
    H --> I["Generate RTSP URL:\ngen.get_rtsp_url\nchannel_id, date, start_time"]
    I --> J["Convert RTSP to HLS proxy URL\nvia getproxyurl\nrtsp_url, request.get_host"]
    J --> K["Return JSON:\n{ rtsp_url: hlsproxyurl }"]
```

**Step-by-step:**

1. **Camera Lookup** — Queries `Camera_model` with a join through `dvr` to match both the DVR host IP and camera name. Extracts `main_stream_id` (the Hikvision channel identifier, e.g., `101`).

2. **DVR Credential Lookup** — Fetches the `DvrInfo` record by host to retrieve `host`, `username`, and `password`.

3. **RTSP URL Generation** — Uses `HikvisionURLGenerator` to build a Hikvision playback RTSP URL:
   ```
   rtsp://<username>:<password>@<dvr_host>:554/Streaming/tracks/<channel_id>?starttime=<YYYYMMDD>T<HHMMSS>Z
   ```

4. **HLS Proxy Conversion** — Calls `getproxyurl(rtsp_url, request.get_host())` to convert the raw RTSP URL into an HLS-compatible HTTP proxy URL that can be consumed by a video player.

### Response

```json
{
  "rtsp_url": "http://<edge_host>:<port>/hls/<stream_session>/index.m3u8"
}
```

> [!IMPORTANT]
> Despite the key name `rtsp_url`, the returned value is actually an **HLS proxy URL** (HTTP), not a raw RTSP URL. The `getproxyurl()` function handles the RTSP-to-HLS transcoding on the edge device.

---

## End-to-End Playback Flow

### Complete User Journey: Playing Back a Recording

```mermaid
sequenceDiagram
    participant U as User / Flutter App
    participant D as Django Backend
    participant E as Edge Device
    participant N as NVR/DVR

    rect rgb(40, 40, 60)
    Note over U,D: Phase 1 — Load Hierarchy
    U->>D: GET /playback/get/ (no device_id)
    D->>D: apply_user_scope() → filter sites, devices, cameras
    D-->>U: { sites: [...], cameras: [...] }
    end

    rect rgb(40, 60, 40)
    Note over U,N: Phase 2 — Scan Available Recordings
    U->>D: GET /playback/get/?device_id=5&dvr_id=12&channel_id=101&action=scan
    D->>E: POST {edge}/scan_storage/ { track_id: 101, dvr_host: "192.168.1.64" }
    E->>N: Query recording index
    N-->>E: Available time ranges
    E-->>D: [{ date, start, end }, ...]
    D-->>U: [{ date, start, end }, ...] (returned as-is)
    end

    rect rgb(60, 40, 40)
    Note over U,N: Phase 3 — Get Playback URL
    U->>D: GET /playback/get/?device_id=5&dvr_id=12&camera_name=Lobby&date=2026-04-09&start_time=14:00:00
    D->>E: GET {edge}/get_playback_url/?dvr_host=192.168.1.64&camera_name=Lobby&date=...&start_time=...
    E->>E: playbackurl() → HikvisionURLGenerator → getproxyurl()
    E-->>D: { rtsp_url: "http://edge:8080/hls/.../index.m3u8" }
    D->>D: Rewrite → /dvr/proxy/?url=http%3A%2F%2Fedge%3A8080%2Fhls%2F...
    D-->>U: { url: "/api/livestream/dvr/proxy/?url=..." }
    end

    rect rgb(60, 50, 30)
    Note over U,N: Phase 4 — Stream Video via dvr_proxy
    U->>D: GET /dvr/proxy/?url=<encoded_m3u8>
    D->>E: GET http://edge:8080/hls/.../index.m3u8
    E-->>D: Raw .m3u8 playlist
    D->>D: Rewrite segment URIs → /dvr/proxy/?url=<segment>
    D-->>U: Rewritten .m3u8
    loop For each video segment
        U->>D: GET /dvr/proxy/?url=<encoded_segment.ts>
        D->>E: Stream segment bytes
        E-->>D: Binary video data
        D-->>U: StreamingHttpResponse (512KB chunks)
    end
    end
```

---

## M3U8 Proxy Rewriting

When `dvr_proxy` receives a request for a `.m3u8` file, it rewrites all internal URLs so that subsequent requests also route through the proxy. This is essential because the client cannot reach the edge device directly.

### Original Playlist (from edge/NVR)
```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-KEY:METHOD=AES-128,URI="key.bin",IV=0x00000001
#EXTINF:10.0,
segment001.ts
#EXTINF:10.0,
segment002.ts
#EXT-X-ENDLIST
```

### Rewritten Playlist (returned to client)
```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-KEY:METHOD=AES-128,URI="/api/livestream/dvr/proxy/?url=http%3A%2F%2F192.168.1.100%3A8080%2Fhls%2Fkey.bin",IV=0x00000001
#EXTINF:10.0,
/api/livestream/dvr/proxy/?url=http%3A%2F%2F192.168.1.100%3A8080%2Fhls%2Fsegment001.ts
#EXTINF:10.0,
/api/livestream/dvr/proxy/?url=http%3A%2F%2F192.168.1.100%3A8080%2Fhls%2Fsegment002.ts
#EXT-X-ENDLIST
```

**Rewriting rules applied line-by-line:**

| Line Type | Example | Action |
|---|---|---|
| Comment with `URI="..."` | `#EXT-X-KEY:...URI="key.bin"` | Inner URI extracted, resolved to absolute, rewritten through proxy |
| Non-comment, non-empty | `segment001.ts` | Resolved to absolute URL via `urljoin()`, rewritten through proxy |
| Other comments / empty | `#EXTINF:10.0,` | Passed through unchanged |

> [!TIP]
> The `urljoin()` call ensures relative paths like `segment001.ts` or `../keys/key.bin` are correctly resolved against the original upstream playlist URL before encoding.

---

## Authentication & Security

| Endpoint | Auth Required | Mechanism |
|---|---|---|
| `getDvrPlayback` | ✅ | `JWTAuthentication` or `SessionAuthentication` |
| `dvr_proxy` | ✅ | `JWTAuthentication` or `SessionAuthentication` |

**RBAC Scoping (Hierarchy Mode):**
- `set_current_user(user)` activates row-level security context
- `apply_user_scope()` filters `EdgeDevice` querysets so users only see authorized sites/devices/cameras

**DVR Credential Flow:**
- DVR credentials (`username`, `password`) are stored in the `DvrInfo` model on the edge device
- They are **never** transmitted to the client
- `HikvisionURLGenerator` embeds them into the RTSP URL on the edge, which is then converted to an HLS proxy URL by `getproxyurl()`

---

## Error Handling

| Scenario | HTTP Status | Response Body |
|---|---|---|
| Missing `url` parameter on `dvr_proxy` | `400` | `"Missing url"` |
| Edge device not found (`device_id` invalid) | `404` | Django's default 404 (via `get_object_or_404`) |
| DVR not found in `DvrInfo` table (edge-side) | `404` | `{ "error": "DVR not found" }` |
| Edge device unreachable or request timeout | `502` | `{ "success": false, "error": "Upstream error: ..." }` |
| Edge response is not valid JSON | `502` | `{ "success": false, "error": "Failed to parse edge response", "raw": "<raw text>" }` |
| Upstream HTTP error (4xx/5xx from NVR/edge) | `502` | `"Upstream error: ..."` |
| Any unhandled exception | `500` | `{ "success": false, "error": "<exception message>" }` |

> [!TIP]
> **Debugging checklist** for playback failures:
> 1. Is the edge device reachable? (check `build_edge_exe_base_url` output)
> 2. Is the NVR/DVR reachable from the edge? (check `DvrInfo.host` / `dvr_host`)
> 3. Does the camera name exist under the correct DVR? (check `Camera_model` + `dvr` relationship)
> 4. Is the date/time within recorded footage? (use `action=scan` first to verify)
> 5. Is the HLS stream healthy? (inspect the raw `.m3u8` content from the edge)
