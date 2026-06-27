> This document was written by AI and has been manually reviewed.

# QQ

NextBridge connects to QQ via the OneBot 11 WebSocket protocol. Multiple protocol backends are supported: [NapCat](https://napneko.github.io) (default), [Lagrange.OneBot](https://github.com/LagrangeDev/Lagrange.Core), and any generic OneBot 11 implementation.

## Setup

1. Install and run your chosen OneBot 11 backend (e.g. NapCat), configured as a WebSocket server.
2. Note the WebSocket URL (default: `ws://127.0.0.1:3001`) and any access token you configured.
3. Add the instance to `data/config.json`.

## Config keys

Add under `qq.<instance_id>` in `config.json`:

| Key | Required | Default | Description |
|---|---|---|---|
| `protocol` | No | `"napcat"` | OneBot 11 backend protocol: `"napcat"`, `"lagrange"`, or `"onebot_v11"`. Controls which protocol-specific features are available (forward API, stream upload, etc.) |
| `ws_url` | No | `ws://127.0.0.1:3001` | WebSocket URL of the OneBot 11 server |
| `ws_token` | No | — | Access token (appended as `?access_token=...`) |
| `ws_ssl_verify` | No | `true` | Whether to verify TLS certificates for WSS connections. Set to `false` for self-signed certs |
| `max_file_size` | No | `10485760` (10 MB) | Maximum bytes to download per attachment when sending |
| `cqface_mode` | No | `"gif"` | How to represent QQ face/emoji segments. `"gif"` uploads the face as an animated GIF (from the local `db/cqface-gif/` database); `"emoji"` renders it as inline text, e.g. `:cqface306:`. |
| `file_send_mode` | No | `"stream"` | How to upload files and videos to QQ. `"stream"` uses chunked `upload_file_stream` (recommended for large files); `"base64"` encodes the whole payload and passes it directly to `upload_group_file`. |
| `stream_threshold` | No | `0` (disabled) | If greater than 0, automatically switches to `"stream"` mode when a file or video exceeds this many bytes, regardless of `file_send_mode`. |
| `forward_render_enabled` | No | `false` | Enable QQ combined-forward message rendering as HTML pages |
| `forward_render_ttl_seconds` | No | `15552000` (180 days) | TTL for rendered forward pages in seconds |
| `forward_render_mount_path` | No | `"/qq-forward"` | HTTP mount path for forward page endpoints |
| `forward_render_persist_enabled` | No | `false` | Persist forward pages to database for survival across restarts |
| `forward_render_image_method` | No | `"url"` | Image rendering method for forward pages: `"url"` (serve via DB+bridge URL) or `"base64"` (inline data URI) |
| `forward_render_asset_ttl_seconds` | No | `1209600` (14 days) | TTL for cached forward page images/assets |
| `forward_render_base_url` | No | — | Custom public URL prefix for forward page links. When set, links are generated as `{base_url}/{page_id}` (mount path is NOT appended automatically) |
| `forward_render_cqface_gif` | No | `true` | Forward face rendering strategy: `false` (unicode emoji), `true` (default gif host), or a custom URL string for the gif host base |
| `proxy` | No | — | Proxy URL for WebSocket connection and media downloading (e.g., `http://proxy.example.com:8080` or `socks5://proxy.example.com:1080`). Set to `null` to explicitly disable proxy for this instance (ignores global proxy setting). |
| `media_proxy` | No | — | Proxy URL used only when fetching media/attachments. Defaults to following `proxy` when unset. |

```json
{
  "qq": {
    "qq_main": {
      "protocol": "napcat",
      "ws_url": "ws://127.0.0.1:3001",
      "ws_token": "your_secret",
      "max_file_size": 10485760
    }
  }
}
```

## Rule channel keys

Use under `channels` or `from`/`to` in `rules.json`:

| Key | Description |
|---|---|
| `group_id` | QQ group number (string or number) |
| `user_id` | QQ user ID for private/DM messages (string or number) |

```json
{
  "qq_main": { "group_id": "947429526" }
}
```

::: info Group and DM messages
NextBridge bridges **group messages** by default. Private (DM) messages can also be bridged by specifying `user_id` instead of `group_id`. The `/nb bind` commands can be used in DMs.
:::

## Message segments

Incoming messages are parsed from OneBot 11 segment arrays:

| Segment type | Handling |
|---|---|
| `text` | Becomes message text |
| `at` | Converted to `@name` text |
| `image` | Forwarded as `image` attachment |
| `record` | Forwarded as `voice` attachment |
| `video` | Forwarded as `video` attachment |
| `file` | Forwarded as `file` attachment |
| Others (face, reply, forward...) | Silently skipped |

## Sending

| Attachment type | Method |
|---|---|
| `image` | Downloaded and sent as base64 (`base64://...`) |
| `voice` | Downloaded and sent as base64 (`base64://...`) |
| `video` | Downloaded and sent via `file_send_mode` (stream or base64) |
| `file` | Downloaded and sent via `file_send_mode` (stream or base64) |

The `file_send_mode` and `stream_threshold` config keys control how videos and files are uploaded. Stream mode (`upload_file_stream` → `upload_group_file`) is the default and handles large files more reliably. Use `"base64"` if stream upload is unsupported by your OneBot backend, and set `stream_threshold` to automatically fall back to stream for files above a given size.

## Combined Forward Rendering

When `forward_render_enabled` is `true`, QQ combined-forward messages are rendered as standalone HTML pages with full media support (images, voice, video, files). The rendered pages are accessible via the HTTP server at the configured `forward_render_mount_path`.

- **Page destruction**: Rendered pages are removed when they are destroyed or expire, so refreshes will return 404 instead of reopening the old page.
- **Page settings**: The forward page includes a top-right settings button for color mode and forward display mode, remembered in LocalStorage.
- **Unreliable sender IDs**: When the OneBot backend cannot reliably map a sender ID inside a combined-forward batch, NextBridge labels the UID as unreliable in the rendered page.
- **Per-rule TTL override**: `forward_render_ttl_seconds` can be overridden per-rule via the `msg` config in rules.

## Notes

- **Self-message echo**: The OneBot backend echoes the bot's own outgoing messages back as events. NextBridge filters these out automatically by comparing `user_id` with `self_id`.
- **Reconnection**: If the WebSocket connection drops, NextBridge automatically reconnects every 5 seconds.
