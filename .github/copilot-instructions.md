# Open WebUI Development Guidelines

## Architecture Overview

Open WebUI is a **dual-stack application** combining SvelteKit (frontend) and FastAPI (backend) with real-time Socket.IO integration. It's designed as a self-hosted AI chat interface supporting multiple LLM providers (Ollama, OpenAI-compatible APIs) with built-in RAG capabilities.

### Key Components

- **Backend** (`backend/open_webui/`): FastAPI app on port 8080
  - Modular routers in `routers/` (auths, chats, models, etc.)
  - Dual model pattern: Pydantic (API contracts) + SQLAlchemy (DB schema)
  - Automatic Peewee migrations on startup (`internal/migrations/`)
- **Frontend** (`src/`): SvelteKit app on port 5173
  - Domain-organized components (`components/chat/`, `components/admin/`)
  - Reactive Svelte stores with Socket.IO integration (`lib/stores/`)
  - API clients mirror backend router structure (`lib/apis/`)
- **Real-time**: Socket.IO server integrated into FastAPI for collaborative features

## Development Workflows

### Essential Commands

**CRITICAL**: Always run `npm run pyodide:fetch` before any frontend build/dev commands. This downloads required Pyodide packages to `static/pyodide/`.

```bash
# Backend development
cd backend && ./dev.sh              # Starts uvicorn on :8080 with CORS for :5173

# Frontend development  
npm run dev                          # Auto-runs pyodide:fetch, starts Vite on :5173

# Linting (unified across stacks)
npm run lint                         # Runs frontend (eslint), types (svelte-check), backend (pylint)
npm run format                       # Prettier for frontend
npm run format:backend               # Black for backend
```

### Development Setup

1. Backend requires Python 3.11+ with dependencies from `backend/requirements.txt`
2. Frontend requires Node.js 18-22.x
3. Backend dev mode sets `CORS_ALLOW_ORIGIN="http://localhost:5173;http://localhost:8080"`
4. Environment variables: See `.env.example` and `backend/open_webui/config.py`

## Code Patterns & Conventions

### Backend Patterns

**Router Structure** (`backend/open_webui/routers/`)
- Each domain gets a dedicated router (e.g., `chats.py`, `users.py`)
- Auth dependency injection via `get_verified_user`, `get_admin_user` from `utils.auth`
- Example from `routers/auths.py`:
  ```python
  router = APIRouter()
  
  @router.post("/signin")
  async def signin(form_data: SigninForm) -> SigninResponse:
      # Router handles domain logic
  ```

**Dual Model Convention**
- API models (Pydantic): `models/auths.py` → `SigninForm`, `SignupForm`
- DB models (SQLAlchemy): Same file → `Auth` class with `__tablename__`
- Migration pattern: New migration files in `internal/migrations/` follow `NNN_description.py`

**Configuration System** (`backend/open_webui/config.py`)
- Hybrid: Database-backed with JSON fallback
- `PersistentConfig` class wraps env vars with DB persistence
- Redis caching for frequently accessed configs

### Frontend Patterns

**Component Organization**
- Domain-based: `components/chat/`, `components/admin/`, `components/workspace/`
- Common/shared: `components/common/` (Sidebar, Modal, etc.)
- Icons: `components/icons/` (separate Svelte components)

**Store Architecture** (`src/lib/stores/index.ts`)
- Centralized reactive stores: `user`, `config`, `models`, `chats`
- Socket.IO store: `socket` writable with real-time event handling
- Example integration:
  ```typescript
  import { user, config, socket } from '$lib/stores';
  // Reactive: $user, $config, $socket in components
  ```

**API Client Pattern** (`src/lib/apis/`)
- Each file mirrors backend router: `apis/chats/`, `apis/auths/`, etc.
- Authentication: Bearer token from `localStorage.token`
- Base URLs from `lib/constants.ts`: `WEBUI_API_BASE_URL`

### Authentication Flow

1. Login via `/api/v1/auths/signin` → returns JWT token
2. Token stored in `localStorage.token`
3. All API calls include `Authorization: Bearer ${token}` header
4. Backend validates via `get_verified_user` dependency

### Real-time Features

Socket.IO requires token authentication:
```javascript
socket.auth = { token: localStorage.token };
socket.connect();
```

Backend Socket.IO handlers in `backend/open_webui/socket/main.py`

## Pipelines Framework

External plugin system via [Pipelines](https://github.com/open-webui/pipelines):
- Run separate Pipelines instance
- Configure via Admin Settings → Connections → OpenAI API URL
- Examples: function calling, rate limiting, usage monitoring, content filtering
- See `backend/open_webui/routers/pipelines.py` for integration

## RAG Integration

Document processing flows through:
- `backend/open_webui/retrieval/` - Embedding and vector search
- Multiple engines: Docling, Tika, Azure Document Intelligence
- Configuration in `backend/open_webui/config.py` (search "DOCLING", "TIKA")

## Database & Migrations

- **ORM**: Peewee (legacy) + SQLAlchemy (modern)
- **Migrations**: Auto-run on startup, managed by `peewee-migrate`
- Add migrations: Create `backend/open_webui/internal/migrations/NNN_description.py`
- Follow existing pattern in `009_add_models.py` for reference

## Testing

- Frontend: `npm run test:frontend` (Vitest)
- E2E: Cypress (`npm run cy:open`)
- Backend: Tests in `backend/open_webui/test/`

## Docker & Deployment

- Primary deployment: Docker Compose (`docker-compose.yaml`)
- Helper scripts: `run-compose.sh`, `Makefile`
- Backend startup: `backend/start.sh` (production) or `dev.sh` (development)
- Environment setup: Admin user via `ADMIN_USER_EMAIL` + `ADMIN_USER_PASSWORD`

## Common Gotchas

1. **Pyodide**: Must run `npm run pyodide:fetch` before builds - downloads Python runtime packages
2. **CORS**: Backend dev mode requires specific CORS config for frontend :5173
3. **Ports**: Backend :8080, Frontend :5173 - don't mix!
4. **Auth**: Backend uses JWT; frontend stores in localStorage.token
5. **Socket.IO**: Requires token auth, not just cookie-based session
