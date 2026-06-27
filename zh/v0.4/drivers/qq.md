> 本文档由 AI 编写，已经人工审核。

# QQ

NextBridge 通过 OneBot 11 WebSocket 协议连接 QQ。支持多种协议后端：[NapCat](https://napneko.github.io)（默认）、[Lagrange.OneBot](https://github.com/LagrangeDev/Lagrange.Core)，以及任何通用 OneBot 11 实现。

## 准备工作

1. 安装并运行你选择的 OneBot 11 后端（如 NapCat），将其配置为 WebSocket 服务端模式。
2. 记录 WebSocket 地址（默认：`ws://127.0.0.1:3001`）和你设置的访问令牌。
3. 在 `data/config.json` 中添加实例配置。

## 配置项

在 `config.json` 的 `qq.<实例ID>` 下添加：

| 键 | 是否必填 | 默认值 | 说明 |
|---|---|---|---|
| `protocol` | 否 | `"napcat"` | OneBot 11 后端协议：`"napcat"`、`"lagrange"` 或 `"onebot_v11"`。控制哪些协议特定功能可用（合并转发 API、流式上传等） |
| `ws_url` | 否 | `ws://127.0.0.1:3001` | OneBot 11 服务端的 WebSocket 地址 |
| `ws_token` | 否 | — | 访问令牌（作为 `?access_token=...` 追加到 URL） |
| `ws_ssl_verify` | 否 | `true` | WSS 连接是否验证 TLS 证书。自签名证书请设为 `false` |
| `max_file_size` | 否 | `10485760`（10 MB） | 发送附件时单个文件的最大下载字节数 |
| `cqface_mode` | 否 | `"gif"` | QQ 表情段的呈现方式。`"gif"` 将表情以动态 GIF 图上传（来自本地 `db/cqface-gif/` 数据库）；`"emoji"` 以内联文本呈现，如 `:cqface306:`。 |
| `file_send_mode` | 否 | `"stream"` | 向 QQ 上传文件和视频的方式。`"stream"` 使用分块 `upload_file_stream`（推荐用于大文件）；`"base64"` 将整个内容编码后直接传给 `upload_group_file`。 |
| `stream_threshold` | 否 | `0`（禁用） | 大于 0 时，当文件或视频超过该字节数时自动切换为 `"stream"` 模式，忽略 `file_send_mode` 的设置。 |
| `forward_render_enabled` | 否 | `false` | 启用 QQ 合并转发消息渲染为 HTML 页面 |
| `forward_render_ttl_seconds` | 否 | `15552000`（180 天） | 渲染的合并转发页面存活时间（秒） |
| `forward_render_mount_path` | 否 | `"/qq-forward"` | 合并转发页面的 HTTP 挂载路径 |
| `forward_render_persist_enabled` | 否 | `false` | 将合并转发页面持久化到数据库，重启后仍可访问 |
| `forward_render_image_method` | 否 | `"url"` | 合并转发页面的图片渲染方式：`"url"`（通过数据库+桥接 URL 提供）或 `"base64"`（内联 data URI） |
| `forward_render_asset_ttl_seconds` | 否 | `1209600`（14 天） | 合并转发页面缓存图片/资源的 TTL |
| `forward_render_base_url` | 否 | — | 合并转发页面链接的自定义公共 URL 前缀。设置后链接格式为 `{base_url}/{page_id}`（不会自动追加挂载路径） |
| `forward_render_cqface_gif` | 否 | `true` | 合并转发表情渲染策略：`false`（unicode 表情）、`true`（默认 gif 主机）或自定义 gif 主机基础 URL 字符串 |
| `proxy` | 否 | — | 用于 WebSocket 连接和附件下载的代理 URL（例如：`http://proxy.example.com:8080` 或 `socks5://proxy.example.com:1080`）。设置为 `null` 可显式禁用此实例的代理（忽略全局代理设置）。 |
| `media_proxy` | 否 | — | 仅用于获取媒体/附件的代理 URL。未设置时默认跟随 `proxy`。 |

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

## 规则频道键

在 `rules.json` 的 `channels` 或 `from`/`to` 下使用：

| 键 | 说明 |
|---|---|
| `group_id` | QQ 群号（字符串或数字均可） |
| `user_id` | QQ 用户 ID，用于私聊消息（字符串或数字均可） |

```json
{
  "qq_main": { "group_id": "947429526" }
}
```

::: info 群消息与私聊消息
NextBridge 默认桥接**群消息**。通过指定 `user_id`（而非 `group_id`）也可以桥接私聊消息。`/nb bind` 指令可在私聊中使用。
:::

## 消息段解析

收到的消息依据 OneBot 11 消息段数组解析：

| 段类型 | 处理方式 |
|---|---|
| `text` | 转为消息文本 |
| `at` | 转为 `@名称` 格式的文本 |
| `image` | 作为 `image` 附件转发 |
| `record` | 作为 `voice` 附件转发 |
| `video` | 作为 `video` 附件转发 |
| `file` | 作为 `file` 附件转发 |
| 其他（表情、回复、合并转发...） | 静默跳过 |

## 发送

| 附件类型 | 发送方式 |
|---|---|
| `image` | 下载后以 base64 编码发送（`base64://...`） |
| `voice` | 下载后以 base64 编码发送（`base64://...`） |
| `video` | 下载后按 `file_send_mode` 发送（stream 或 base64） |
| `file` | 下载后按 `file_send_mode` 发送（stream 或 base64） |

`file_send_mode` 和 `stream_threshold` 配置项控制视频和文件的上传方式。Stream 模式（`upload_file_stream` → `upload_group_file`）为默认值，对大文件更可靠。如果你的 OneBot 后端不支持流式上传，可改为 `"base64"`；配置 `stream_threshold` 可在文件超过指定大小时自动回退到 stream 模式。

## 合并转发渲染

当 `forward_render_enabled` 为 `true` 时，QQ 合并转发消息会被渲染为独立的 HTML 页面，支持完整的媒体内容（图片、语音、视频、文件）。渲染页面可通过 HTTP 服务器在配置的 `forward_render_mount_path` 路径访问。

- **页面销毁**：合并转发页面在销毁或过期后会直接失效，刷新时会返回 404，不会再次打开旧页面。
- **页面设置**：合并转发页面右上角提供设置入口，可切换颜色模式与合并转发显示方式，并使用 LocalStorage 记忆。
- **不可靠 UID 标记**：当 OneBot 后端在同一批合并转发中无法可靠对应发送者 ID 时，页面会把该 UID 标记为不可靠。
- **按规则覆盖 TTL**：`forward_render_ttl_seconds` 可通过规则中的 `msg` 配置按规则覆盖。

## 注意事项

- **自身消息回显**：OneBot 后端会将机器人自己发送的消息作为真实事件回传。NextBridge 通过对比 `user_id` 与 `self_id` 自动过滤这类消息。
- **自动重连**：WebSocket 连接断开后，NextBridge 每隔 5 秒自动重新连接。
