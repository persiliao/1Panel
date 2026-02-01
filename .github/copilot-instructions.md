# 1Panel AI Agent Instructions

## Project Overview
1Panel is a web-based Linux server management control panel with a **Go backend** (agent + core) and **TypeScript/Vue 3 frontend**. The project follows a layered architecture: API → Service → Repository → Model.

## Architecture

### Backend Structure (Go)
- **`agent/`**: Main backend service serving `/api/v2` endpoints
  - `cmd/server/main.go`: Entry point (Cobra CLI framework)
  - `app/api/v2/`: API handlers (request/response, ~50 router files)
  - `app/service/`: Business logic (database operations, external integrations)
  - `app/repo/`: Data access layer using GORM
  - `app/model/`: Database models
  - `app/dto/`: Request/response structs with `validate` tags
  - `app/task/`: Async task system with rollback support
  - `init/`: Application initialization (db, router, cache, logger, config)
  - `utils/`: Helpers for Docker, nginx, SSH, file ops, etc.
  - `middleware/`: Auth (certificate validation for agents), i18n
  - `global/`: Singleton services (DB, logger, validator, cron scheduler)

- **`core/`**: Web dashboard backend (similar structure to `agent/`)
  - Manages UI auth, settings, operation logs
  - Proxies requests to agent via HTTP

### Frontend Structure (TypeScript/Vue 3)
- **`frontend/src/`**:
  - `api/`: HTTP client wrapper with interceptors, request signing
  - `views/`: Page components (Dashboard, App Store, Container, File Manager, etc.)
  - `components/`: Reusable UI components
  - `store/`: Pinia state management
  - `routers/`: Vue Router navigation
  - `lang/`: i18n translations (20+ languages)

## Critical Patterns

### API Handler Pattern
All handlers follow this structure in [agent/app/api/v2/](agent/app/api/v2/):
```go
func (b *BaseApi) OperationName(c *gin.Context) {
    // 1. Parse & validate request using helper.CheckBindAndValidate()
    var req request.RequestType
    if err := helper.CheckBindAndValidate(&req, c); err != nil {
        return  // Helper returns error response automatically
    }
    
    // 2. Call service method
    result, err := serviceInstance.Method(req)
    if err != nil {
        helper.InternalServer(c, err)  // Consistent error response
        return
    }
    
    // 3. Return response
    helper.SuccessWithData(c, result)
}
```
See [agent/app/api/v2/file.go](agent/app/api/v2/file.go) or [agent/app/api/v2/website.go](agent/app/api/v2/website.go) for real examples.

### Response Format
All API responses use standard envelope in [agent/app/dto/common_res.go](agent/app/dto/common_res.go):
```go
type Response struct {
    Code    int         `json:"code"`      // HTTP status code
    Message string      `json:"message"`   // i18n error key or success msg
    Data    interface{} `json:"data"`      // Response payload
}
```

### Error Handling
- **BusinessError** type in [agent/buserr/errors.go](agent/buserr/errors.go) wraps i18n-aware error messages
- Use `buserr.New("I18nKeyName")` for localized errors
- Helper functions: `helper.InternalServer(c, err)`, `helper.BadRequest(c, err)`
- All i18n keys defined in `agent/i18n/lang/` and `core/i18n/lang/`

### Service Initialization
[agent/server/server.go](agent/server/server.go) startup order is critical:
1. `viper.Init()` - Load config from env/config.yaml
2. `dir.Init()` - Create required directories
3. `db.Init()` - Connect to SQLite/MySQL/PostgreSQL
4. `app.Init()` - Register business services
5. `cron.Run()` - Start scheduled jobs

Services are registered as singletons in [agent/global/global.go](agent/global/global.go).

### Async Task System
[agent/app/task/task.go](agent/app/task/task.go) handles long-running operations with:
- **SubTask**: Individual action with retry/timeout/rollback
- **ActionFunc**: Execute logic, **RollbackFunc**: Undo on failure
- Task context stored in `global.TaskCtxMap` for cancellation
- Useful for: app installs, backups, database operations

### Router Registration
Routes in [agent/router/common.go](agent/router/common.go) group endpoints:
- Each resource has a dedicated router (e.g., `AppRouter`, `ContainerRouter`)
- Registered in `RouterGroups()` and initialized in [agent/init/router/router.go](agent/init/router/router.go)
- API base path: `/api/v2`

### Frontend API Client
[frontend/src/api/index.ts](frontend/src/api/index.ts) provides:
- `RequestHttp` class with interceptors for token signing and error handling
- Response codes: `200` (success), `401` (auth), `400` (validation), `500` (server)
- Global loading state via Pinia store during long operations
- Token format: MD5 hash of `'1panel' + API-Key + Timestamp`

## Development Workflows

### Build Frontend
```bash
cd frontend
npm install
npm run build:pro   # Production build
npm run dev         # Dev server on port (see vite.config.ts)
```

### Build Backend
```bash
# From workspace root
make build_core_on_linux      # Build 1panel-core
make build_agent_on_linux     # Build 1panel-agent
make build_all                # Both + frontend
```

### Run Locally
- Agent listens on port 9999 (configurable)
- Core listens on port 9998 (configurable)
- Frontend proxy: `/api/v2` → `http://localhost:9999/` (vite.config.ts)

### Database
- Uses **GORM** ORM for MySQL/PostgreSQL/SQLite
- Migrations in `agent/init/migration/`
- Repo methods follow interface pattern: `type IAppRepo interface { ... }`
- Query helpers: `repo.WithByKey()`, `repo.WithOrderBy()`

## Project-Specific Conventions

1. **Error Keys**: All error messages are i18n keys (e.g., `"ErrInvalidParams"`, `"ErrInternalServer"`). Define in language files, not hardcoded strings.

2. **DTO Validation**: Request structs in `app/dto/request/` use `binding:` tags:
   ```go
   type InstallAppReq struct {
       AppID   string `json:"appId" binding:"required"`
       Version string `json:"version" binding:"required"`
   }
   ```

3. **Service Interfaces**: Services are always defined as interfaces (e.g., `IAppService`) for testability. Implementations registered in `entry.go`.

4. **Context Cancellation**: Long operations accept `context.Context` for graceful shutdown. Store cancel functions in `global.TaskCtxMap`.

5. **Logging**: Use `global.LOG` (logrus) consistently. Format: `global.LOG.Errorf("message: %v", err)`

6. **Config**: App config in `agent/global/config.go`, loaded via `agent/init/viper/viper.go` from config.yaml or env vars.

7. **WebSocket**: Terminal/file tail streams use gorilla/websocket. See `agent/utils/websocket/` for helpers.

8. **Docker Integration**: Standardized helpers in `agent/utils/docker/` for container operations, compose management via [compose-spec](https://github.com/compose-spec/compose-go).

## Key Files Reference

| Purpose | File |
|---------|------|
| API entry point | [agent/app/api/v2/entry.go](agent/app/api/v2/entry.go) |
| Error definitions | [agent/buserr/errors.go](agent/buserr/errors.go) |
| Global state | [agent/global/global.go](agent/global/global.go) |
| Task system | [agent/app/task/task.go](agent/app/task/task.go) |
| Helper functions | [agent/app/api/v2/helper/helper.go](agent/app/api/v2/helper/helper.go) |
| Frontend HTTP client | [frontend/src/api/index.ts](frontend/src/api/index.ts) |
| Vite config | [frontend/vite.config.ts](frontend/vite.config.ts) |
| Build commands | [Makefile](Makefile) |

## Before Asking for Help
- Check existing service implementations in `agent/app/service/` (e.g., `app_utils.go`)
- Review similar API handlers in `agent/app/api/v2/`
- Look for model/DTO examples in related domain files
- Run `grep -r "type.*Service struct"` to find services
