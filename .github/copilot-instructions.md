# Open WebUI Copilot Instructions

## Project Architecture

This is a **SvelteKit frontend + FastAPI backend** chat interface for LLMs, supporting Ollama and OpenAI-compatible APIs.

### Key Structure
- **Backend**: `backend/open_webui/` - FastAPI application with modular routers
- **Frontend**: `src/` - SvelteKit app with TypeScript
- **Models**: SQLAlchemy ORM with Alembic migrations
- **Real-time**: Socket.IO for live features (chat streaming, user presence)

## Development Workflows

### Local Development
```bash
# Backend (runs on :8080)
cd backend && ./dev.sh

# Frontend (runs on :5173) 
npm run dev
```

### Key Commands
- `npm run pyodide:fetch` - Required before frontend builds (fetches Python runtime)
- `npm run lint` - Runs frontend, types, and backend linting
- `make install` - Docker Compose setup for full stack

## Backend Patterns

### Router Structure
- Each domain has its own router in `backend/open_webui/routers/`
- Example: `chats.py`, `users.py`, `models.py`
- All routes use dependency injection for auth: `user=Depends(get_verified_user)`

### Database Models
- **Convention**: Separate Pydantic models from SQLAlchemy models
- **Models dir**: `backend/open_webui/models/` contains both DB schema and Pydantic forms
- **Example**: `ChatForm` (request), `ChatResponse` (response), `Chat` (SQLAlchemy table)

### Configuration System
- **Dual storage**: Database config + JSON file fallback
- **Pattern**: All config in `backend/open_webui/config.py` with environment variables
- **Redis**: Optional caching layer with `get_redis_connection()`

## Frontend Patterns

### Component Organization
```
src/lib/components/
├── chat/          # Chat-specific UI
├── admin/         # Admin panels  
├── common/        # Reusable components
└── app/           # App shell components
```

### Store Management
- **Global stores**: `src/lib/stores/index.ts` - includes `user`, `config`, `chats`, `models`
- **Socket integration**: Real-time updates via `socket` store
- **Reactive patterns**: Use `$:` for derived state, avoid manual subscriptions

### API Integration
- **Base pattern**: All APIs in `src/lib/apis/` mirror backend router structure
- **Auth**: Bearer token in `localStorage.token`, passed to all requests
- **Error handling**: Standardized error responses with toast notifications

## Key Integrations

### Socket.IO Usage
- **Backend**: `backend/open_webui/socket/main.py` - handles events
- **Frontend**: Connected via `src/routes/+layout.svelte` 
- **Pattern**: Real-time chat updates, user presence, model status

### Function/Tool System
- **Backend**: `backend/open_webui/functions.py` - Python function execution
- **Frontend**: Tools workspace in `src/lib/components/workspace/`
- **Integration**: Custom tools via Pipelines framework

### Multi-Model Support
- **Ollama**: Direct integration via `/api/ollama` proxy
- **OpenAI**: Compatible API support with custom base URLs
- **Pattern**: Model abstraction in `src/lib/stores` handles both types uniformly

## Critical File Dependencies

### Configuration Flow
1. `backend/open_webui/env.py` - Environment setup
2. `backend/open_webui/config.py` - Runtime configuration  
3. `src/lib/constants.ts` - Frontend constants

### Authentication Chain
1. `backend/open_webui/utils/auth.py` - JWT handling
2. `backend/open_webui/routers/auths.py` - Auth endpoints
3. `src/lib/apis/auths.ts` - Frontend auth API

When modifying auth, update both backend utilities AND frontend API calls.

## Common Gotchas

- **CORS**: Backend dev script sets specific origins - update `backend/dev.sh` for new ports
- **Database migrations**: Run automatically on startup, but check `backend/open_webui/migrations/`
- **Pyodide**: Frontend Python runtime requires `npm run pyodide:fetch` before builds
- **Socket auth**: Requires `localStorage.token` to be set for real-time features