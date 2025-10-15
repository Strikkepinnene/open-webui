# Open WebUI - AI Coding Agent Instructions

## Architecture Overview

Open WebUI is a **SvelteKit frontend + FastAPI backend** application with real-time **Socket.IO** integration. It's a self-hosted AI chat interface supporting multiple LLM providers (Ollama, OpenAI-compatible APIs) with built-in RAG capabilities.

### Service Boundaries
- **Frontend**: SvelteKit (dev: `:5173`, prod served by backend)
- **Backend**: FastAPI (`:8080`) with SQLAlchemy + Alembic migrations
- **Real-time**: Socket.IO with optional Redis manager for clustering
- **Storage**: PostgreSQL/SQLite for data, Redis for caching/sessions

## Critical Development Workflows

### Setup & Running
```bash
# Backend (runs on :8080)
cd backend && ./dev.sh
# Sets CORS_ALLOW_ORIGIN="http://localhost:5173;http://localhost:8080"

# Frontend (runs on :5173) - MUST run pyodide fetch first
npm run pyodide:fetch  # Downloads Python runtime packages
npm run dev

# Unified Linting
npm run lint  # Runs frontend + types + backend linting
npm run lint:backend  # pylint backend/
npm run lint:frontend  # eslint . --fix
```

### Build Process
**CRITICAL**: Always run `npm run pyodide:fetch` before any frontend build. This downloads Pyodide packages (micropip, numpy, pandas, etc.) to `static/pyodide/`. Builds will fail without this.

## Project-Specific Conventions

### Backend Patterns

#### Modular Router Pattern
Each domain has its own router in `backend/open_webui/routers/`:
```python
from fastapi import APIRouter, Depends
from open_webui.utils.auth import get_verified_user, get_admin_user

router = APIRouter()

@router.get("/", response_model=list[ModelResponse])
async def get_items(user=Depends(get_verified_user)):
    # Dependency injection for auth
    return Items.get_by_user_id(user.id)
```

#### Dual Model Convention
Separate models for API contracts and database:
- `backend/open_webui/models/{domain}.py`: SQLAlchemy models + Pydantic forms
- Forms: `{Domain}Form` (create), `{Domain}UpdateForm` (update)
- Responses: `{Domain}Response`, `{Domain}UserResponse` (with access control)

Example from `models/folders.py`:
```python
class FolderForm(BaseModel):  # API input
    name: str
    parent_id: Optional[str] = None

class FolderModel(Base):  # Database model
    __tablename__ = "folder"
    id = Column(String, primary_key=True)
    # ...
```

#### Authentication
All protected routes use `Depends(get_verified_user)` or `Depends(get_admin_user)`. Token extraction happens in `utils/auth.py`.

#### Database Migrations
Automatic on startup via `config.py:run_migrations()`. Create migrations:
```bash
cd backend
alembic revision --autogenerate -m "description"
```

### Frontend Patterns

#### API Client Structure
Frontend APIs in `src/lib/apis/{domain}/index.ts` mirror backend routers exactly:
```typescript
export const getItems = async (token: string) => {
    return fetch(`${WEBUI_API_BASE_URL}/items`, {
        headers: { authorization: `Bearer ${token}` }
    });
};
```

#### Authentication
- Token stored in `localStorage.token`
- All API calls include `authorization: Bearer ${token}` header
- Socket.IO requires token in auth payload

#### Svelte Store Architecture
Centralized reactive stores in `src/lib/stores/index.ts`:
- `config`, `user`, `models`, `chats`, `folders`, `knowledge`, etc.
- `socket` store for Socket.IO connection
- Real-time updates via Socket.IO events

#### Component Organization
```
src/lib/components/
├── admin/          # Admin-only features
├── chat/           # Chat interface components
├── common/         # Shared components (Modal, Tooltip, etc.)
├── icons/          # SVG icon components
├── layout/         # Layout wrappers
└── workspace/      # Tools, functions, knowledge workspaces
```

### Socket.IO Integration
**Critical**: Real-time features require token authentication:
```typescript
// Connect with token
socket = io(WEBUI_BASE_URL, {
    auth: { token: localStorage.token },
    path: '/socket.io'
});
```

Backend authentication in `socket/main.py` via `decode_token()` from auth header.

### Configuration System
Hybrid approach:
1. Database-backed config in `backend/open_webui/config.py` (Config table)
2. Redis caching for performance (optional)
3. Environment variables for secrets/overrides
4. Automatic migration from JSON to database on startup

## Integration Points

### RAG & Knowledge
- Files uploaded via `routers/files.py` → processed by `routers/retrieval.py`
- Vector embeddings stored in configurable vector DB (Chroma, Milvus, etc.)
- Knowledge bases link files to collections via `models/knowledge.py`

### Pipelines Framework
External Python plugins system via `routers/pipelines.py`:
- Upload `.py` files with `Pipe` class
- Inlet/outlet filters for request/response processing
- Enables custom logic, rate limiting, translation, etc.

### LLM Providers
- `routers/ollama.py`: Direct Ollama integration
- `routers/openai.py`: OpenAI-compatible API proxy
- Model definitions in `models/models.py` with access control

## Common Pitfalls

1. **Pyodide Missing**: Build fails → Run `npm run pyodide:fetch`
2. **CORS Errors**: Ensure `dev.sh` sets correct CORS origins
3. **Auth Failures**: Check `localStorage.token` exists and is valid JWT
4. **Migration Issues**: Database out of sync → Automatic migration runs on backend start
5. **Socket.IO Disconnects**: Token expired or not passed in auth payload

## File References

- Entry Points: `backend/open_webui/main.py`, `src/routes/+layout.svelte`
- Auth Logic: `backend/open_webui/utils/auth.py`
- Database Models: `backend/open_webui/models/*.py`
- API Routers: `backend/open_webui/routers/*.py`
- Frontend APIs: `src/lib/apis/*/index.ts`
- Stores: `src/lib/stores/index.ts`
- Socket Handler: `backend/open_webui/socket/main.py`
