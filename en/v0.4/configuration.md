> This document was written by AI and has been manually reviewed.

# Configuration Reference

## Config file formats

NextBridge supports **JSON**, **YAML**, and **TOML** config files. Place the file in the data directory (default: `data/`). The first file found in this order is used:

1. `config.json`
2. `config.yaml` / `config.yml`
3. `config.toml`

### Converting between formats

Use the built-in convert command to translate between formats:

```sh
uv run main.py convert data/config.json data/config.yaml
uv run main.py convert data/config.yaml data/config.toml
```

### Validating configuration

Use the built-in validate command to check syntax correctness without starting the bridge:

```sh
# Validate using default data path
uv run main.py validate

# Specify custom data directory
uv run main.py validate -d my-data/

# Validate specific config file only
uv run main.py validate -c path/to/config.yaml

# Validate specific rules file only
uv run main.py validate -r path/to/rules.json

# Validate both with explicit paths
uv run main.py validate --config config.yaml --rules rules.yaml
```

Exit codes:
- **0** — validation passed
- **1** — syntax/format error found
- **2** — manually specified file not found

## Structure

The config has a two-level structure regardless of format:

```jsonc
{
  "global": {
    // ... global config ...
  },
  "<platform>": {
    "<instance_id>": {
      // ... driver config ...
    }
  }
}
```

| Level | Description |
|---|---|
| `global` | Global configuration options that apply to all drivers unless overridden |
| `<platform>` | One of `qq`, `discord`, `telegram`, `feishu`, `dingtalk`, `yunhu`, `kook`, `vocechat`, `matrix`, `signal`, `teams`, `googlechat`, `slack`, `mattermost`, `rocketchat`, `webhook` |
| `<instance_id>` | A name you choose freely — used to reference this instance in rules |

## Global Configuration

The `global` section contains configuration options that apply to all drivers unless overridden in the driver-specific configuration.

| Key | Required | Default | Description |
|---|---|---|---|
| `proxy` | No | — | Global proxy URL for all drivers that ***support proxy configuration*** (e.g., `http://proxy.example.com:8080`). Individual driver proxy settings will override this global setting. |
| `strict_echo_match` | No | `false` | Controls how NextBridge prevents echoing messages back to the same channel/instance. When `false` (default), skips if target_id == msg.instance_id OR target_channel == msg.channel. When `true`, skips only if target_id == msg.instance_id AND target_channel == msg.channel. Default is `false` to maximize echo prevention. |
| `fuzzy_mention_match` | No | `false` | Controls whether mentions without exact bind mapping should fall back to fuzzy nickname matching. When `true`, attempts to match mentioned user's name against known display names in the target platform. When `false` (default), only exact ID bounds or native platform mentions work. Default is `false`. |
| `command_prefix` | No | `"nb"` | Prefix for built-in bridge commands (without leading `/`). Changes `/nb bind` to `/<prefix> bind`. |
| `base_url` | No | — | Public base URL for generating externally reachable links (e.g., QQ forward page URLs). Example: `https://bridge.example.com` |
| `log` | No | — | Logging configuration for controlling log output and rotation. See [Logging Configuration](#logging-configuration) below. |
| `database` | No | — | Database configuration for storing message and user mappings. See [Database Configuration](#database-configuration) below. |
| `http` | No | — | Shared HTTP server configuration for mounted driver webhooks. See [HTTP Server Configuration](#http-server-configuration) below. |
| `plugins` | No | — | Plugin discovery and driver lifecycle configuration. See [Plugin Configuration](#plugin-configuration) below. |
| `middleware` | No | — | Message middleware configuration. See [Middleware Configuration](#middleware-configuration) below. |

```json
{
  "global": {
    "proxy": "http://proxy.example.com:8080"
  }
}
```

::: tip Using proxy from environment variables
 If not set, the program will attempt to read proxy configuration from environment variables `http_proxy`, `https_proxy`, and `all_proxy` (case-insensitive). You can disable the use of system proxy by setting `proxy` to the special value `disabled`.
:::

## Logging Configuration

NextBridge uses loguru for logging, which supports flexible log output and rotation. The logging configuration controls both console output and file logging behavior.

### Configuration Options

| Key | Required | Default | Description |
|---|---|---|---|
| `log.level` | No | `INFO` | Console log verbosity level. |
| `log.file_level` | No | `DEBUG` | File log verbosity level. Default is DEBUG for verbose output during troubleshooting. |
| `log.dir` | No | `null` | Directory path for log files. If `null` or not specified, file logging is disabled. Log files are automatically created with timestamp-based names in the specified directory. |
| `log.rotation_size` | No | `100 MB` | Maximum size of a single log file before rotation (e.g., `"100 MB"`, `"500 MB"`). Log files are automatically rotated when they exceed this size. |
| `log.retention_days` | No | `7` | Number of days to keep log files. Older log files are automatically deleted. Set to `0` to disable automatic deletion. |
| `log.compression` | No | `zip` | Compression format for rotated log files (e.g., `"zip"`, `"gz"`, `"tar.gz"`). Set to `null` to disable compression. |
| `log.show_source` | No | `"auto"` | Controls when source file locations are shown in logs. `"auto"`: show only for DEBUG/TRACE. `"always"`: always show. `"never"`: never show. |

### Configuration Examples

**Basic logging (console only):**
```json
{
  "global": {
    "log": {
      "level": "INFO"
    }
  }
}
```

**File logging with rotation:**
```json
{
  "global": {
    "log": {
      "level": "INFO",
      "dir": "logs",
      "rotation_size": "100 MB",
      "retention_days": 7,
      "compression": "zip"
    }
  }
}
```

**Verbose logging for debugging:**
```json
{
  "global": {
    "log": {
      "level": "DEBUG",
      "dir": "logs",
      "rotation_size": "50 MB",
      "retention_days": 3,
      "compression": null
    }
  }
}
```

::: tip Log Rotation
  When `log.dir` is set, log files are automatically rotated when they exceed `log.rotation_size`. Old log files are automatically deleted after `log.retention_days` days. Rotated files can be compressed using `log.compression` to save disk space.
:::

## Database Configuration

NextBridge uses SQLAlchemy for database operations, which supports multiple database backends including SQLite, MySQL, PostgreSQL, and more. The database is used to store:

- Message ID mappings between different platforms
- User bindings for cross-platform user identification
- User display name mappings
- Temporary binding codes

### Configuration Options

| Key | Required | Default | Description |
|---|---|---|---|
| `database.url` | No | `sqlite:///data.db` | SQLAlchemy database URL. Relative SQLite paths are resolved under the `data/` directory. |
| `database.echo` | No | `false` | Enable SQLAlchemy query logging for debugging. |
| `database.pool_size` | No | — | Connection pool size for non-SQLite databases. Uses SQLAlchemy default if not specified. |
| `database.max_overflow` | No | — | Maximum overflow size of the pool for non-SQLite databases. Uses SQLAlchemy default if not specified. |
| `database.pool_recycle` | No | `3600` | Recycle connections after this many seconds (default: 1 hour). |

### Database URL Examples

**SQLite (default):**
```json
{
  "global": {
    "database": {
      "url": "sqlite:///data.db"
    }
  }
}
```

**MySQL:**
```json
{
  "global": {
    "database": {
      "url": "mysql+pymysql://user:password@localhost:3306/nextbridge",
      "pool_size": 10,
      "max_overflow": 20,
      "pool_recycle": 3600
    }
  }
}
```

**PostgreSQL:**
```json
{
  "global": {
    "database": {
      "url": "postgresql://user:password@localhost:5432/nextbridge",
      "pool_size": 10,
      "max_overflow": 20,
      "pool_recycle": 3600
    }
  }
}
```

::: tip SQLite Path Handling
  When using SQLite with a relative path (e.g., `sqlite:///data.db`), the path is resolved relative to the data directory (`data/`). Absolute paths (e.g., `sqlite:////var/lib/nextbridge/data.db`) are used as-is.
:::

::: warning Connection Pooling
  Connection pool settings (`pool_size`, `max_overflow`, `pool_recycle`) only apply to non-SQLite databases. SQLite uses a file-based storage and doesn't support connection pooling in the same way.
:::

## HTTP Server Configuration

Several drivers (DingTalk, Feishu, VoceChat, Yunhu, etc.) receive messages via HTTP webhooks. NextBridge runs a shared HTTP server (FastAPI/uvicorn) that mounts all webhook endpoints.

| Key | Required | Default | Description |
|---|---|---|---|
| `http.host` | No | `"0.0.0.0"` | Host/IP for the shared HTTP server |
| `http.port` | No | `9080` | Port for the shared HTTP server |
| `http.root_path` | No | `""` | ASGI `root_path`, useful behind path-prefixed reverse proxies |
| `http.log_level` | No | `"info"` | Uvicorn log level (`critical`, `error`, `warning`, `info`, `debug`) |
| `http.enable` | No | `"unset"` | HTTP server startup mode. `"unset"`: auto start when a driver mounts a sub-app. `"true"`: always start. `"false"`: never start. |

```json
{
  "global": {
    "http": {
      "host": "0.0.0.0",
      "port": 9080
    }
  }
}
```

## Plugin Configuration

Controls driver plugin discovery and lifecycle management.

| Key | Required | Default | Description |
|---|---|---|---|
| `plugins.paths` | No | `[]` | Local directories to scan for driver plugin `.py` files |
| `plugins.auto_restart` | No | `true` | Automatically restart crashed drivers with exponential backoff |
| `plugins.max_restart_attempts` | No | `5` | Maximum restart attempts before a crashed driver is abandoned |
| `plugins.health_check_interval` | No | `60` | Seconds between periodic driver health checks. Set to `0` to disable |
| `plugins.admin.enable` | No | `false` | Enable admin API endpoints (`/_nextbridge/drivers`, `/_nextbridge/admin/reload/{id}`). Requires `password` to be set |
| `plugins.admin.password` | No | `""` | Password for admin API access (HTTP Basic Auth). Required when `admin.enable` is `true` |

```json
{
  "global": {
    "plugins": {
      "auto_restart": true,
      "max_restart_attempts": 5,
      "admin": {
        "enable": true,
        "password": "your-secret-password"
      }
    }
  }
}
```

## Middleware Configuration

Message middleware allows transforming or filtering messages before they are sent. Middleware is evaluated in list order.

| Key | Required | Default | Description |
|---|---|---|---|
| `middleware.enabled` | No | `[]` | Middleware names to enable (evaluated in list order) |

```json
{
  "global": {
    "middleware": {
      "enabled": ["example-middleware"]
    }
  }
}
```

You can run **multiple instances of the same platform** by adding more keys under the platform:

```json
{
  "discord": {
    "server_a": { "bot_token": "...", "webhook_url": "..." },
    "server_b": { "bot_token": "...", "webhook_url": "..." }
  }
}
```

## Full example (JSON)

```json
{
  "qq": {
    "qq_main": {
      "ws_url": "ws://127.0.0.1:3001",
      "ws_token": "secret"
    }
  },
  "discord": {
    "dc_main": {
      "send_method": "webhook",
      "webhook_url": "https://discord.com/api/webhooks/ID/TOKEN",
      "bot_token": "BOT_TOKEN",
      "max_file_size": 8388608
    }
  },
  "telegram": {
    "tg_main": {
      "bot_token": "123456:ABC-DEF",
      "max_file_size": 52428800
    }
  },
  "feishu": {
    "fs_main": {
      "app_id": "cli_xxxx",
      "app_secret": "xxxx",
      "verification_token": "xxxx",
      "encrypt_key": "",
      "listen_port": 8080,
      "listen_path": "/event"
    }
  },
  "dingtalk": {
    "dt_main": {
      "app_key": "dingxxxx",
      "app_secret": "xxxx",
      "robot_code": "xxxx",
      "signing_secret": "xxxx",
      "listen_port": 8082,
      "listen_path": "/dingtalk/event"
    }
  },
  "matrix": {
    "mx_main": {
      "homeserver": "https://matrix.org",
      "user_id": "@mybot:matrix.org",
      "password": "your_password"
    }
  },
  "signal": {
    "sg_main": {
      "api_url": "http://localhost:8080",
      "number": "+12025551234"
    }
  },
  "slack": {
    "sl_main": {
      "bot_token": "xoxb-...",
      "app_token": "xapp-..."
    }
  }
}
```

## Full example (YAML)

```yaml
qq:
  qq_main:
    ws_url: ws://127.0.0.1:3001
    ws_token: secret

discord:
  dc_main:
    send_method: webhook
    webhook_url: https://discord.com/api/webhooks/ID/TOKEN
    bot_token: BOT_TOKEN
    max_file_size: 8388608

matrix:
  mx_main:
    homeserver: https://matrix.org
    user_id: "@mybot:matrix.org"
    password: your_password
```

For per-driver config keys, see the individual driver pages in the [Drivers](./drivers/) section.
