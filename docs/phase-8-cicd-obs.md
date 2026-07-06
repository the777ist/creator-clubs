# Phase 8 — CI/CD, observability, push, realtime, E2E & docs

**Goal:** Close the platform loop. Wire the GitHub Actions workflows that make the
monorepo deployable trunk-based (affected-only CI, Fly API deploys, EAS build + OTA,
nightly E2E + visual regression, Electron releases); add the **observability** spine
(structlog JSON logs + a `request_id` middleware in FastAPI, an `X-Request-Id` injected
by the `packages/core` API-client wrapper, Sentry init on both sides tagged with that id
→ client→API→logs traceability); template the **push notification loop** (token
registration → `/v1/push-tokens` → `send_push()` via the Expo Push API); ship the
canonical **broadcast-only realtime** pattern (API broadcasts an invalidation event on a
per-product channel after a mutation, `packages/core` subscribe-and-invalidate wires the
channel event into TanStack `invalidateQueries`); add one **scheduled job** (a Fly machine
running `tasks.py` to prune stale push tokens); stand up the **E2E harness** (Playwright
signup → login → items CRUD → realtime, plus Storybook visual-regression baselines, both
in `e2e-nightly.yml`, plus one local Maestro flow); and complete the **docs / agent
surface** (README + CLAUDE.md + `.claude/commands/` at root, `packages/ui`, and product).

**Verify (restated from the Phase 8 row):**
> push branch → CI green; touch one product → other is cache-hit; stale `openapi.json`
> fails drift check; items list refreshes across two open clients after a mutation; API
> log lines carry the `request_id`; `e2e-nightly.yml` green via `workflow_dispatch` (E2E +
> visual regression); scheduled task runs via `fly machine run` (push registration needs a
> dev build — Expo Go can't receive push tokens; verified later on real devices).

> **Naming/placeholder reminder (from PHILOSOPHY.md header):** package scope `@platform/*`; bundle
> ids `com.example.*`; infra `<org>-<product>-<env>` with org placeholder `example`; product
> token `template`; clearly-marked placeholders are `example`, `com.example.*`,
> `TODO-EAS-PROJECT-ID`, the releases-repo owner. Every repo/org value in the YAML below is a
> placeholder — swap when real infra accounts exist. Release tags are EXACTLY
> `<product>-<surface>-v*` (surface ∈ api/app/desktop) and the OTA tag is `<product>-ota-v*`;
> EAS Update channels are EXACTLY `staging` / `production`.

---

## Prerequisites

Phase 8 is the capstone — it assumes **Phases 1–7 are complete** and both products exist:

1. **Phase 1** — root tooling: `mise.toml` (Node 24 LTS / pnpm 11 / Python 3.13 / uv),
   `pnpm-workspace.yaml` (`nodeLinker: hoisted` — pnpm 11 relocates this out of `.npmrc`),
   `.npmrc` (auth/registry only), `turbo.json` (2.9 `tasks`),
   `tsconfig.base.json`, `lefthook.yml`, `packages/config`. `turbo run lint` is a clean
   no-op.
2. **Phase 2** — `packages/ui` (owned react-native-reusables primitives, theme infra,
   **Storybook** workbench with theme/brand toolbar + per-variant `*.stories.tsx`),
   `packages/core` (query client + persistence, env), `_template/app` shell.
3. **Phase 3** — `_template/api`: strict layered OOP (`models/` → `services/` → `schemas/`
   → `routers/`), problem+json, cursor pagination, `security.py`, `middleware.py` stub,
   `/healthz` + `/v1/hello` + `/v1/items` CRUD, `db.py`, `auth.py`, Alembic initial
   migration (RLS deny-all), `seed.py`, polyfactory factories, Dockerfile, fly tomls,
   pytest against real Postgres.
4. **Phase 4** — typegen: `export_openapi.py`, `api-client/` (hey-api), turbo wiring,
   `features/home` list via the generated `useInfiniteQuery` hook.
5. **Phase 5** — desktop: `app://` protocol shell, electron-builder.yml, updater wired
   (no-op without a releases repo).
6. **Phase 6** — Supabase local + auth: core session store + guards, `features/auth`
   login/signup, protected `/v1/me`, `core/storage.ts` + avatar upload.
7. **Phase 7** — generator + stamped **`demo`** product (portIndex 1). Both products build
   under `--affected`; `pnpm bootstrap` runs both local stacks; `git grep -iw template
   products/demo` is empty.

This guide **adds files** to `_template` (token-rewritten into `demo` by re-running the
generator, or by the patterns being copied forward); it does **not** re-stamp `demo`.
Several files from earlier phases are **extended here** (notably `api/middleware.py`,
`api/sentry.ts` ↔ `core/sentry.ts`, `core/api.ts`, `core/realtime.ts`, `core/notifications.ts`,
`api/tasks.py`, `api/models/`, `api/routers/`, `api/services/`) — when a file already exists
from an earlier phase the step says so.

---

## Definition of done

- [ ] **Observability:** `api/.../middleware.py` assigns a `request_id` per request
      (reads inbound `X-Request-Id`, else generates one), binds it into a structlog
      contextvar, echoes it back in the `X-Request-Id` response header, and emits **JSON**
      access logs carrying it. `core/api.ts` generates+injects `X-Request-Id` on every
      request. Sentry is initialised on both sides and **tags events with the request id**.
- [ ] **Push loop:** `core/notifications.ts` registers an Expo push token and POSTs it to
      `/v1/push-tokens`; `api/.../models/push_token.py` + `routers/push.py` +
      `PushService.send_push()` (httpx → Expo Push API) exist; `test_push.py` passes with a
      **mocked httpx transport**.
- [ ] **Realtime broadcast-only:** `ItemService` broadcasts an invalidation event on the
      per-product channel (service-role HTTP to Supabase) after every items mutation;
      `core/realtime.ts` subscribes and calls `queryClient.invalidateQueries`; the home
      list is wired to it. **No Postgres-Changes subscriptions, no RLS holes.**
- [ ] **Scheduled job:** `api/.../tasks.py` exposes a `prune_push_tokens` entrypoint
      runnable as `python -m template_api.tasks prune-push-tokens`; documented as a Fly
      scheduled machine (`fly machine run … --schedule`).
- [ ] **E2E harness:** `products/_template/app/playwright.config.ts` + `app/e2e/*.spec.ts`
      (signup → login → items CRUD → realtime) run against exported `dist` + local API +
      Supabase local. A Storybook VR Playwright script iterates `storybook-static/index.json`
      and screenshots each story × {light,dark}; baselines committed. One `.maestro/` flow
      exists (local only).
- [ ] **Workflows:** `ci.yml`, `deploy-api.yml`, `eas-build.yml`, `eas-update.yml`,
      `e2e-nightly.yml`, `electron-release.yml` all present in `.github/workflows/`, valid
      YAML (actionlint-clean), using clearly-marked placeholders. **Plus the per-product
      `eas.json` (channels EXACTLY `staging`/`production`) and `vercel.json` (SPA rewrite)**
      — in PHILOSOPHY's tree but created by no earlier phase (step f).
- [ ] **Docs/agent surface:** root `CLAUDE.md` + `README.md`; `packages/ui/CLAUDE.md` +
      `FIGMA.md`; product `CLAUDE.md` + `README.md` (+ nested api CLAUDE.md recipe);
      `.claude/commands/` at all three levels with the documented command inventories.
- [ ] All **Verification** commands pass.

---

## Build steps

> Run from repo root unless noted. Paths are repo-relative. `<product>` token in `_template`
> is the literal `template`; the generator rewrites it.

### (a) Observability — request_id, structlog JSON, Sentry both sides, X-Request-Id

**Files**
- `products/_template/api/src/template_api/middleware.py` *(extend — created as a stub in
  Phase 3)*
- `products/_template/api/src/template_api/logging.py` *(new — structlog config)*
- `products/_template/api/src/template_api/sentry.py` *(new — server Sentry init)*
- `products/_template/api/src/template_api/main.py` *(extend — register middleware + init)*
- `packages/core/src/api.ts` *(extend — X-Request-Id injection)*
- `packages/core/src/sentry.ts` *(extend — tag request id)*
- `products/_template/app/app.config.ts` *(extend — add the `@sentry/react-native/expo`
  config plugin; cross-reference Phase 2 step (h))*
- `products/_template/app/metro.config.ts` *(extend — compose `getSentryExpoConfig` with
  `withNativeWind`; created in Phase 2)*

**Contents**

`api/.../logging.py` — structlog rendering JSON, sharing stdlib's stream:
```python
import logging
import sys
import structlog

request_id_var: structlog.contextvars  # documented contextvar key: "request_id"

def configure_logging(*, level: str = "INFO") -> None:
    logging.basicConfig(format="%(message)s", stream=sys.stdout, level=level)
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,   # pulls request_id into every line
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.dict_tracebacks,
            structlog.processors.JSONRenderer(),        # JSON logs
        ],
        # getLevelNamesMapping (3.11+): the str->int direction of logging.getLevelName is
        # deprecated and fails pyright strict.
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.getLevelNamesMapping()[level.upper()]
        ),
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

log = structlog.get_logger()
```

`api/.../middleware.py` — request id + structlog binding + access log:
```python
import time
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.types import ASGIApp

REQUEST_ID_HEADER = "X-Request-Id"
log = structlog.get_logger()

class RequestIdMiddleware(BaseHTTPMiddleware):
    """Bind a request_id for the lifetime of the request: honour an inbound
    X-Request-Id (set by the core api-client wrapper) or mint a UUIDv4, expose it
    on structlog contextvars + Sentry scope, echo it on the response."""

    def __init__(self, app: ASGIApp) -> None:
        super().__init__(app)

    async def dispatch(self, request: Request, call_next):
        rid = request.headers.get(REQUEST_ID_HEADER) or uuid.uuid4().hex
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(request_id=rid)
        # tag Sentry so server-side events carry the same id (client→API→logs)
        try:
            import sentry_sdk
            sentry_sdk.set_tag("request_id", rid)
        except Exception:  # Sentry optional in local/dev
            pass
        request.state.request_id = rid
        start = time.perf_counter()
        response = await call_next(request)
        response.headers[REQUEST_ID_HEADER] = rid
        log.info(
            "http_request",
            method=request.method,
            path=request.url.path,
            status=response.status_code,
            duration_ms=round((time.perf_counter() - start) * 1000, 2),
        )
        return response
```

`api/.../sentry.py`:
```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.starlette import StarletteIntegration
from .settings import settings   # SENTRY_DSN, ENV (staging|production), RELEASE

def init_sentry() -> None:
    if not settings.SENTRY_DSN:
        return  # no-op locally / in CI without a DSN
    sentry_sdk.init(
        dsn=settings.SENTRY_DSN,
        environment=settings.ENV,
        release=settings.RELEASE,
        integrations=[FastApiIntegration(), StarletteIntegration()],
        traces_sample_rate=0.1,
        send_default_pii=False,
    )
```

`api/.../main.py` (excerpt — order matters, request-id middleware **outermost** so it wraps
errors too):
```python
configure_logging(level=settings.LOG_LEVEL)
init_sentry()
app = FastAPI(...)
# security middleware (CORS/headers/slowapi) from Phase 3 added first (inner),
# request-id added LAST so it is the outermost layer:
app.add_middleware(RequestIdMiddleware)
```

`packages/core/src/api.ts` (extend the hey-api client wrapper — inject id + auth header).
**The client is INJECTED, never imported by name:** `packages/core` is shared and never
stamped, so `import { client } from "@platform/<product>-api-client"` is impossible there —
the `<product>` token can't be rewritten in a shared package, and a hard import of the
template's client ships a latent cross-product bug (a stamped product's `_layout.tsx`
silently configures the TEMPLATE's client singleton while its own hooks use its own,
unconfigured client — this actually shipped in Phases 4–7 before the audit caught it):
```ts
import { captureRequestId } from "./sentry";

/** Structural shape of the generated hey-api client — keeps core product-agnostic. */
export type GeneratedApiClient = {
  setConfig: (config: { baseUrl?: string }) => unknown;
  interceptors: {
    request: { use: (fn: (request: Request) => Request | Promise<Request>) => unknown };
  };
};

// crypto.randomUUID exists on web + Hermes (RN 0.85). Fallback kept for safety.
function newRequestId(): string {
  return typeof crypto !== "undefined" && "randomUUID" in crypto
    ? crypto.randomUUID()
    : `${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

/**
 * Each product calls this from ITS OWN app/_layout.tsx with ITS OWN generated client:
 *   import { client } from "@platform/template-api-client";
 *   configureApiClient(client, { baseUrl: env.apiUrl, getToken: getAccessToken });
 */
export function configureApiClient(
  client: GeneratedApiClient,
  opts: { baseUrl: string; getToken: () => string | null },
) {
  client.setConfig({ baseUrl: opts.baseUrl });
  client.interceptors.request.use((request) => {
    const rid = newRequestId();
    request.headers.set("X-Request-Id", rid);
    captureRequestId(rid);              // tag the client-side Sentry scope
    const token = opts.getToken();
    if (token) request.headers.set("Authorization", `Bearer ${token}`);
    return request;
  });
}
```

`packages/core/src/sentry.ts` (extend — note `@sentry/react-native`, NOT `sentry-expo`):
```ts
import * as Sentry from "@sentry/react-native";
import { env } from "./env";

export function initSentry() {
  // Field names follow env.ts's committed camelCase accessor shape (env.apiUrl, ...) —
  // add `sentryDsn` / `appEnv` there reading EXPO_PUBLIC_SENTRY_DSN / EXPO_PUBLIC_ENV.
  if (!env.sentryDsn) return; // no-op without DSN
  Sentry.init({
    dsn: env.sentryDsn,
    environment: env.appEnv,               // staging | production
    tracesSampleRate: 0.1,
  });
}

export function captureRequestId(requestId: string) {
  Sentry.setTag("request_id", requestId); // matches the API tag → traceable
}
```
> `Sentry.init()` alone is NOT enough for production source maps / native symbolication — it
> ships the JS-only runtime half. For Expo you ALSO need the build-time half: the config
> plugin (below) + Metro wiring (below). Pin `@sentry/react-native` to a release that lists
> **Expo SDK 56 / RN 0.85** support.

`app/app.config.ts` (extend — add the Expo config plugin; cross-reference the Phase 2
`app.config.ts` that already sets `scheme`, bundle ids, `extra.eas.projectId`, and the
`updates.url` + `runtimeVersion` OTA policy):
```ts
// inside the Expo config `plugins` array:
plugins: [
  // ...existing plugins (expo-router, etc.)
  [
    "@sentry/react-native/expo",
    {
      // organization/project + auth token enable source-map upload at build time.
      // SENTRY_AUTH_TOKEN is a BUILD env var (EAS secret) — never committed.
      organization: "example",      // PLACEHOLDER org slug
      project: "example-template",  // PLACEHOLDER Sentry project slug
    },
  ],
],
```

`app/metro.config.ts` (extend — compose Sentry's Metro config with NativeWind; created in
Phase 2 with `getDefaultConfig` + `withNativeWind`):
```ts
const { getSentryExpoConfig } = require("@sentry/react-native/metro");
const { withNativeWind } = require("nativewind/metro");

// Sentry FIRST (replaces getDefaultConfig), THEN wrap with NativeWind.
const config = getSentryExpoConfig(__dirname);
config.watchFolders = [workspaceRoot];           // ../../.. — preserve Phase 2 monorepo wiring
config.resolver.nodeModulesPaths = [
  projectRoot + "/node_modules",
  workspaceRoot + "/node_modules",
];
module.exports = withNativeWind(config, { input: "./global.css" });
```
> ⚠️ REVIEW: Phase 2's `metro.config.js` uses `getDefaultConfig`; here it is swapped for
> `getSentryExpoConfig(__dirname)` (which internally calls `getDefaultConfig` and adds the
> Sentry serializer). Keep the existing `watchFolders`/`nodeModulesPaths` monorepo wiring.

**Commands**
```bash
# structlog + sentry-sdk[fastapi] are ALREADY Phase 3 deps (PHILOSOPHY's dep list) — the
# uv add is a no-op kept for idempotence:
cd products/_template/api && uv add structlog "sentry-sdk[fastapi]" && cd -
# BEFORE this install: add `"@sentry/cli": true` to pnpm-workspace.yaml allowBuilds —
# its postinstall downloads the sentry-cli binary (pnpm 11 blocks it otherwise).
pnpm --filter @platform/core add @sentry/react-native
# the Expo config plugin + Metro helper ship inside the same package — no extra install.
turbo run typecheck --filter=*template-api --filter=@platform/core
```

**Why** — PHILOSOPHY.md Observability: "Sentry + structlog JSON logs + request_id middleware; the
API-client wrapper sends a generated X-Request-Id per request; Sentry events tagged with it
on both sides → client→API→logs traceability." `@sentry/react-native` is the locked SDK
(`sentry-expo` is deprecated; pin a release listing Expo SDK 56 / RN 0.85 support). On Expo,
`Sentry.init()` is only the runtime half — production source maps and native symbolication
need the `@sentry/react-native/expo` **config plugin** in `app.config.ts` PLUS `getSentryExpoConfig`
**Metro** wiring composed with `withNativeWind` (and `SENTRY_AUTH_TOKEN` as an EAS build secret,
never committed). The request-id middleware must be outermost so even error responses carry
the id; structlog `merge_contextvars` is what threads the id into every log line emitted during
the request.

---

### (b) Push loop — register → /v1/push-tokens → send_push()

> **RECONCILIATION (2026-07-05 run): most of the server side ALREADY SHIPPED IN PHASE 3** —
> the model (`models/push_token.py`, per **user+device** with `uq_push_user_device`, per the
> gospel), `schemas/push.py`, `PushService` (`services/push_service.py`), `routers/push.py`,
> and the RLS deny-all migration. **Phase 3's shapes are authoritative** — where the
> skeletons below differ (file names, a per-user+token constraint, DTO field names), keep
> Phase 3's. Phase 8's REAL additions: `test_push.py`, the core registration helper, the app
> wiring, and the `tasks.py` alignment (step d).

**Files**
- `packages/core/src/notifications.ts` *(extend — created stub in Phase 2/6 tree)*
- `products/_template/api/src/template_api/models/push_token.py` *(verify — Phase 3)*
- `products/_template/api/src/template_api/schemas/push.py` *(verify — Phase 3)*
- `products/_template/api/src/template_api/services/push_service.py` *(verify — Phase 3)*
- `products/_template/api/src/template_api/routers/push.py` *(verify — Phase 3)*
- `products/_template/api/alembic/versions/0001_initial.py` *(verify — Phase 3's initial
  migration already creates `push_token` with RLS deny-all)*
- `products/_template/api/tests/test_push.py` *(new)*

**Contents**

`core/notifications.ts` — same injection rule as `api.ts`: shared core can NEVER import a
product's generated SDK by name, so the product passes its generated call in:
```ts
import * as Notifications from "expo-notifications";
import * as Device from "expo-device";

/** The product's generated SDK call for POST /v1/push-tokens, injected at the call site:
 *    import { registerToken } from "@platform/template-api-client";
 *    registerForPushNotifications((body) => registerToken({ body }));
 */
export type RegisterPushToken = (body: {
  device_id: string;
  expo_token: string;
}) => Promise<unknown>;

export async function registerForPushNotifications(
  post: RegisterPushToken,
): Promise<string | null> {
  if (!Device.isDevice) return null; // simulators/Expo Go cannot receive a token
  const { status: existing } = await Notifications.getPermissionsAsync();
  let status = existing;
  if (existing !== "granted") status = (await Notifications.requestPermissionsAsync()).status;
  if (status !== "granted") return null;
  const { data: token } = await Notifications.getExpoPushTokenAsync();
  await post({ device_id: Device.osInternalBuildId ?? "unknown", expo_token: token });
  return token;
}
```
> Body field names follow Phase 3's `PushTokenCreate` DTO (`device_id` + `expo_token`, the
> per-user+device shape) — not an ad-hoc `{token, platform}` pair.

`api/.../models/push_token.py` (SQLModel, UUIDv7 base from Phase 3):
```python
from sqlmodel import Field, UniqueConstraint
from .base import UUIDBase   # UUIDv7 PK base from Phase 3

class PushToken(UUIDBase, table=True):
    __tablename__ = "push_token"
    __table_args__ = (UniqueConstraint("user_id", "token", name="uq_push_user_token"),)
    user_id: str = Field(index=True)        # Supabase auth uid
    token: str                              # ExponentPushToken[...]
    platform: str = "unknown"
```

`api/.../schemas/push.py`:
```python
from pydantic import BaseModel, ConfigDict

class PushTokenIn(BaseModel):
    model_config = ConfigDict(strict=True)
    token: str
    platform: str = "unknown"

class PushTokenOut(BaseModel):
    id: str
    token: str
    platform: str
```

`api/.../services/push.py` (service holds the session; httpx for the external call):
```python
import httpx
import structlog
from sqlmodel import Session, select
from ..models.push_token import PushToken
from .base import BaseService

log = structlog.get_logger()
EXPO_PUSH_URL = "https://exp.host/--/api/v2/push/send"

class PushService(BaseService):
    def register(self, *, user_id: str, token: str, platform: str) -> PushToken:
        existing = self.session.exec(
            select(PushToken).where(PushToken.user_id == user_id, PushToken.token == token)
        ).first()
        if existing:
            return existing
        row = PushToken(user_id=user_id, token=token, platform=platform)
        self.session.add(row)
        self.session.commit()
        self.session.refresh(row)
        return row

    async def send_push(self, *, user_id: str, title: str, body: str,
                        http: httpx.AsyncClient | None = None) -> None:
        tokens = self.session.exec(
            select(PushToken.token).where(PushToken.user_id == user_id)
        ).all()
        if not tokens:
            return
        messages = [{"to": t, "title": title, "body": body} for t in tokens]
        client = http or httpx.AsyncClient(timeout=10.0)
        try:
            resp = await client.post(EXPO_PUSH_URL, json=messages)
            resp.raise_for_status()
            log.info("push_sent", count=len(messages))
        finally:
            if http is None:
                await client.aclose()
```
> `http` is injectable so `test_push.py` passes an `httpx.AsyncClient` backed by
> `httpx.MockTransport` (mocking conventions: API unit tests mock external HTTP via httpx
> mock transport).

`api/.../routers/push.py` (thin; depends on the service + `CurrentUser`):
```python
from fastapi import APIRouter, Depends
from ..auth import CurrentUser
from ..schemas.push import PushTokenIn, PushTokenOut
from ..services.push import PushService

router = APIRouter(prefix="/v1/push-tokens", tags=["push"])

@router.post("", response_model=PushTokenOut, status_code=201)
def register_token(body: PushTokenIn, user: CurrentUser,
                   svc: PushService = Depends(PushService)) -> PushTokenOut:
    row = svc.register(user_id=user.id, token=body.token, platform=body.platform)
    return PushTokenOut(id=str(row.id), token=row.token, platform=row.platform)
```

`tests/test_push.py` (excerpt):
```python
import httpx, pytest
from template_api.services.push import PushService

@pytest.mark.asyncio
async def test_send_push_mocks_expo(session, user):
    PushService(session).register(user_id=user.id, token="ExponentPushToken[x]", platform="ios")
    seen = {}
    def handler(req: httpx.Request) -> httpx.Response:
        seen["body"] = req.content
        return httpx.Response(200, json={"data": [{"status": "ok"}]})
    transport = httpx.MockTransport(handler)
    async with httpx.AsyncClient(transport=transport) as http:
        await PushService(session).send_push(user_id=user.id, title="hi", body="yo", http=http)
    assert b"ExponentPushToken" in seen["body"]
```

**Commands**
```bash
# httpx is already a Phase 3 runtime dep; the push_token table + RLS deny-all already exist
# in Phase 3's initial migration — NO new migration is needed here.
pnpm --filter @platform/template-app exec expo install expo-notifications expo-device
cd products/_template/api && uv run pytest tests/test_push.py && cd -
turbo run openapi --filter=*template-api   # idempotent — the push endpoint is already in the contract
```

**Why** — PHILOSOPHY.md: "Push notifications: full loop templated — token registration in the app
(expo-notifications), `/v1/push-tokens` endpoint + table (per user+device), `send_push()`
service calling Expo's Push API via httpx." Phase 3's initial migration already applies
**RLS deny-all** on `push_token` (DB convention: every table RLS deny-all; the API's
privileged role bypasses it). Expo Go cannot receive a token — registration only works in a
dev build (gotcha below).

---

### (c) Realtime broadcast-only — API broadcast + core subscribe-and-invalidate

**Files**
- `products/_template/api/src/template_api/services/realtime.py` *(new — broadcast helper)*
- `products/_template/api/src/template_api/services/items.py` *(extend — broadcast on mutation)*
- `packages/core/src/realtime.ts` *(extend — subscribe-and-invalidate)*
- `products/_template/app/features/home/*` *(extend — wire the subscription)*

**Contents**

`api/.../services/realtime.py` — broadcast via Supabase Realtime's HTTP broadcast endpoint
using the **service role** key (tables stay RLS-locked; we never open Postgres-Changes):
```python
import httpx
import structlog
from ..settings import settings   # SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY

log = structlog.get_logger()

async def broadcast_invalidate(resource: str, *, http: httpx.AsyncClient | None = None) -> None:
    """Tell clients on the per-product channel to invalidate `resource`.
    Channel name is product-scoped: `<product>:realtime`. No DB subscription opened."""
    url = f"{settings.SUPABASE_URL}/realtime/v1/api/broadcast"
    payload = {"messages": [{
        "topic": "<product>:realtime",
        "event": "invalidate",
        "payload": {"resource": resource},
    }]}
    headers = {
        "apikey": settings.SUPABASE_SERVICE_ROLE_KEY,
        "Authorization": f"Bearer {settings.SUPABASE_SERVICE_ROLE_KEY}",
    }
    client = http or httpx.AsyncClient(timeout=5.0)
    try:
        resp = await client.post(url, json=payload, headers=headers)
        resp.raise_for_status()
        log.info("broadcast", resource=resource)
    finally:
        if http is None:
            await client.aclose()
```
> ✅ RESOLVED (local, 2026-07-05): the server-side path is CONFIRMED live —
> `POST {SUPABASE_URL}/realtime/v1/api/broadcast` with `apikey` + service-role bearer returns
> **202 Accepted** against the local stack (CLI current as of 2026-07), and the E2E proves
> end-to-end delivery (a second client repaints via broadcast → invalidate → refetch).
> Re-verify once a HOSTED project exists (Supabase has iterated on Realtime auth / RLS on
> `realtime.messages`); the architecture does not change either way.

`api/.../services/items.py` (extend the create/update/delete paths — broadcast after commit):
```python
from .realtime import broadcast_invalidate

class ItemService(BaseService):
    async def create(self, data: ItemCreate) -> Item:
        row = Item(**data.model_dump())
        self.session.add(row); self.session.commit(); self.session.refresh(row)
        await broadcast_invalidate("items")   # clients refetch through the API
        return row
    # update() / delete() call broadcast_invalidate("items") after their commit too
```
> Router methods that call these become `async def` and `await` the service — the broadcast
> is fire-and-don't-block-correctness; failures are logged, not fatal (catch in the service
> or let it raise per product policy). ⚠️ OPEN / TO CONFIRM: PHILOSOPHY.md does not pin whether a
> broadcast failure should fail the mutation — default here is "log + swallow" so a Realtime
> outage never breaks writes; confirm per product.

`packages/core/src/realtime.ts` (the shipped subscribe-and-invalidate helper).
**hey-api query keys are OBJECT-shaped** — `[{ _id: "listItems", baseUrl, ... }]` — so a bare
`invalidateQueries({ queryKey: ["items"] })` NEVER matches a generated query. The helper
therefore takes a `keys` map (resource → generated key-fn results); TanStack's partial deep
matching then also catches the `_infinite` variant. Unmapped resources still fall back to
`[resource]` for hand-written queries:
```ts
import type { QueryClient, QueryKey } from "@tanstack/react-query";
import type { SupabaseClient } from "@supabase/supabase-js";

// Wires channel `invalidate` events → TanStack invalidation. No Postgres-Changes.
export function subscribeAndInvalidate(
  supabase: SupabaseClient,
  queryClient: QueryClient,
  opts: {
    channel: string;                       // e.g. "<product>:realtime"
    /** resource → generated query keys, e.g. { items: [listItemsQueryKey()] } */
    keys?: Record<string, QueryKey[]>;
  },
) {
  const channel = supabase
    .channel(opts.channel)
    .on("broadcast", { event: "invalidate" }, (msg) => {
      const resource = (msg.payload as { resource?: string }).resource;
      if (!resource) return;
      const mapped = opts.keys?.[resource];
      if (mapped) {
        for (const queryKey of mapped) void queryClient.invalidateQueries({ queryKey });
      } else {
        void queryClient.invalidateQueries({ queryKey: [resource] }); // hand-written queries
      }
    })
    .subscribe();
  return () => {
    void supabase.removeChannel(channel);
  };
}
```

`app/features/home/*` (wire it — call inside the home screen's effect; the product supplies
ITS OWN generated key fns — shared core never imports them):
```ts
import { useEffect } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { supabase, subscribeAndInvalidate } from "@platform/core";
import { listItemsQueryKey } from "@platform/template-api-client";

export function useItemsRealtime() {
  const queryClient = useQueryClient();
  useEffect(
    () =>
      subscribeAndInvalidate(supabase, queryClient, {
        channel: "<product>:realtime",
        keys: { items: [listItemsQueryKey()] },
      }),
    [queryClient],
  );
}
```

**Commands**
```bash
turbo run typecheck --filter=@platform/core --filter=*template-api
turbo run test --filter=*template-api      # service tests can mock broadcast httpx transport
```

**Why** — PHILOSOPHY.md Realtime (canonical, locked): "broadcast-only — tables stay RLS-locked;
after mutations FastAPI broadcasts invalidation events on per-product channels (service-role
HTTP call); clients refetch through the API. `packages/core` ships the subscribe-and-invalidate
helper. No Postgres-Changes subscriptions, no RLS holes, schema stays private." The channel is
**per-product** (`<product>:realtime`) so two products never cross-talk.

---

### (d) Scheduled job — prune stale push tokens via a Fly machine

**Files**
- `products/_template/api/src/template_api/tasks.py` *(extend — stub in Phase 3 tree)*
- `products/_template/api/fly.staging.toml` / `fly.production.toml` *(reference only — the
  scheduled machine is created via CLI, documented here)*

**Contents**

`api/.../tasks.py` — a tiny CLI module (no queue infra; one example task):
```python
"""Scheduled jobs run as one-off Fly machines: `python -m template_api.tasks <task>`."""
import sys
from datetime import datetime, timedelta, timezone
import structlog
from sqlmodel import Session, delete
from .db import engine
from .logging import configure_logging
from .models.push_token import PushToken

log = structlog.get_logger()
STALE_AFTER = timedelta(days=90)

def prune_push_tokens() -> int:
    cutoff = datetime.now(timezone.utc) - STALE_AFTER
    with Session(engine) as session:
        # SQLModel's session.exec() only types select(); delete()/update() MUST go through
        # SQLAlchemy's session.execute() — exec(delete(...)) fails pyright strict and lacks
        # .rowcount. execute(delete(...)) returns a Result whose .rowcount is valid.
        result = session.execute(delete(PushToken).where(PushToken.updated_at < cutoff))
        session.commit()
        count = result.rowcount or 0
    log.info("pruned_push_tokens", count=count)
    return count

TASKS = {"prune-push-tokens": prune_push_tokens}

def main() -> None:
    configure_logging()
    if len(sys.argv) != 2 or sys.argv[1] not in TASKS:
        raise SystemExit(f"usage: python -m template_api.tasks <{'|'.join(TASKS)}>")
    TASKS[sys.argv[1]]()

if __name__ == "__main__":
    main()
```
> Requires an `updated_at` column on `PushToken` (add to the UUIDv7 base or the model). ⚠️
> OPEN / TO CONFIRM: PHILOSOPHY.md says "prune stale push tokens" but does not define "stale" —
> 90 days by last-update is a documented default; confirm per product.

**Commands** (run against the staging Fly app — `<org>` placeholder):
```bash
# one-off run (the Verify step):
fly machine run \
  --app example-template-api-stg \
  registry.fly.io/example-template-api-stg:latest \
  python -m template_api.tasks prune-push-tokens

# scheduled (daily) machine — Fly's built-in scheduler, no queue infra:
fly machine run \
  --app example-template-api-stg \
  --schedule daily \
  registry.fly.io/example-template-api-stg:latest \
  python -m template_api.tasks prune-push-tokens
```
> `fly machine run --schedule` takes **interval keywords ONLY** —
> `hourly` / `daily` / `weekly` / `monthly` (NOT cron expressions) — and runs are **"fuzzy"**
> (approximate, not guaranteed at an exact minute). `daily` is fine for prune-push-tokens. If
> a product ever needs precise cron (e.g. `0 4 * * *`), use Fly's **Cron Manager** (per-job
> isolated machines) or **Supercronic** per Fly's task-scheduling blueprint — `--schedule`
> alone won't do it. The one-off Verify run (no `--schedule`) is a correct one-shot.

**Why** — PHILOSOPHY.md Background/scheduled jobs: "Fly scheduled machines running a lightweight
`tasks` module in the api (no queue infra); template ships one example (prune stale push
tokens)." The task module reuses the API's `engine` + structlog config so logs are JSON and
land in the same place. `--schedule` is Fly's native (keyword-based, fuzzy) scheduler — exact
cron needs Cron Manager / Supercronic.

---

### (e) E2E harness — Playwright web E2E + Storybook visual regression + Maestro

**Files**
- `products/_template/app/playwright.config.ts` *(new)*
- `products/_template/app/e2e/items.spec.ts` *(new — signup → login → CRUD → realtime)*
- `products/_template/app/e2e/global-setup.ts` *(new — build dist, start API + supabase)*
- `packages/ui/.storybook/visual-regression.spec.ts` *(new — VR over storybook-static)*
- `packages/ui/playwright.config.ts` *(new — VR project)*
- `products/_template/app/.maestro/login.yaml` *(new — local mobile flow)*

**Contents**

`app/playwright.config.ts` — two hard-won rules baked in: (1) **ports derive from
`product.json` at config load** (`8000 + 10·portIndex` / `54321 + 100·portIndex`) — the
"ports come from product.json" doctrine applies to EVERY file the generator copies, not just
env/config files; hardcoded 8000/54321 here made the first stamped product's E2E hit the
TEMPLATE's stack; (2) **both long-lived processes are Playwright `webServer` entries**
(multi-webServer) — Playwright owns readiness + teardown, global-setup only prepares state:
```ts
import { defineConfig, devices } from "@playwright/test";
import { readFileSync } from "node:fs";
import { join } from "node:path";

const { portIndex } = JSON.parse(
  readFileSync(join(__dirname, "..", "product.json"), "utf8"),
) as { portIndex: number };
export const API_PORT = 8000 + 10 * portIndex;
export const SUPABASE_PORT = 54321 + 100 * portIndex;

export default defineConfig({
  testDir: "./e2e",
  globalSetup: "./e2e/global-setup.ts",
  timeout: 60_000,
  use: { baseURL: "http://localhost:8081", trace: "on-first-retry" },
  webServer: [
    {
      // `-s` (SPA fallback) is REQUIRED — without it deep links like /signup 404.
      command: "npx serve dist -s -l 8081",
      url: "http://localhost:8081",
      reuseExistingServer: !process.env.CI,
    },
    {
      command: `uv run --project ../api uvicorn template_api.main:app --port ${API_PORT}`,
      url: `http://localhost:${API_PORT}/healthz`,
      reuseExistingServer: !process.env.CI,
    },
  ],
  projects: [{ name: "chromium", use: devices["Desktop Chrome"] }],
});
```

`app/e2e/items.spec.ts` (skeleton — full-stack flow). Selector realities from the built
screens: **auth inputs use PLACEHOLDERS, not labels** → `getByPlaceholder`, and the Phase 6
login button reads **"Sign in"**, not "Log in":
```ts
import { test, expect } from "@playwright/test";

test("signup → login → items CRUD → realtime", async ({ browser }) => {
  const a = await browser.newContext();
  const pageA = await a.newPage();
  const email = `e2e+${Date.now()}@example.com`;

  await pageA.goto("/signup");
  await pageA.getByPlaceholder("Email").fill(email);
  await pageA.getByPlaceholder("Password").fill("Passw0rd!");
  await pageA.getByRole("button", { name: "Sign up" }).click();
  await expect(pageA.getByRole("tab", { name: "Home" })).toBeVisible();

  // create an item
  await pageA.getByRole("button", { name: "Add item" }).click();
  await pageA.getByPlaceholder("Title").fill("first item");
  await pageA.getByRole("button", { name: "Save" }).click();
  await expect(pageA.getByText("first item")).toBeVisible();

  // realtime: a SECOND client sees the next mutation without manual refresh.
  // NOTE (persisted-cache design): context B seeded from storageState REHYDRATES client A's
  // persisted query cache and (fresh enough) won't refetch on mount — asserting the
  // PRE-broadcast "first item" is visible in B FAILS BY DESIGN. The realtime proof is the
  // POST-broadcast item appearing (its refetch also pulls the older item in).
  const b = await browser.newContext({ storageState: await a.storageState() });
  const pageB = await b.newPage();
  await pageB.goto("/");
  await pageA.getByRole("button", { name: "Add item" }).click();
  await pageA.getByPlaceholder("Title").fill("broadcast item");
  await pageA.getByRole("button", { name: "Save" }).click();
  await expect(pageB.getByText("broadcast item")).toBeVisible({ timeout: 10_000 });
});
```

`app/e2e/global-setup.ts` — STATE PREPARATION ONLY (the long-lived processes are the
config's `webServer` entries): supabase up-check, migrate, seed, export. Two export traps
are handled explicitly: (1) **`expo export` pins `NODE_ENV=production`**, so
`.env.production`'s placeholders win over `.env.development` — parse `.env.development` and
inject its `EXPO_PUBLIC_*` as DIRECT env vars (they beat dotenv); (2) **Metro's transform
cache doesn't key on `EXPO_PUBLIC_*`** — export with `--clear` or the previous bundle
replays byte-identical. Also sanitize `CI`: expo-cli's `getenv.boolish("CI")` THROWS on an
empty-string `CI=`:
```ts
import { execSync } from "node:child_process";
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { API_PORT, SUPABASE_PORT } from "../playwright.config";

export default async function globalSetup() {
  // 1. local Supabase must be up (per-product offset ports from config.toml)
  execSync(`curl -sf http://localhost:${SUPABASE_PORT}/rest/v1/ -o /dev/null || (echo "supabase not up" && exit 1)`, { stdio: "inherit", shell: "bash" });
  // 2. migrate + seed
  execSync("uv run alembic upgrade head && uv run python -m template_api.seed", {
    cwd: join(__dirname, "..", "..", "api"),
    stdio: "inherit",
  });
  // 3. export the web bundle: .env.development values injected as DIRECT env vars + --clear
  const envFile = readFileSync(join(__dirname, "..", ".env.development"), "utf8");
  const publicVars = Object.fromEntries(
    envFile.split("\n")
      .filter((l) => l.startsWith("EXPO_PUBLIC_"))
      .map((l) => l.split("=", 2) as [string, string]),
  );
  const env = { ...process.env, ...publicVars, NODE_ENV: "development" };
  if (env.CI === "") delete env.CI; // getenv.boolish("CI") throws on empty string
  execSync("npx expo export --platform web --clear", {
    cwd: join(__dirname, ".."),
    stdio: "inherit",
    env,
  });
}
```
> ✅ RESOLVED (was OPEN — process orchestration): both long-lived processes (`serve -s dist`,
> uvicorn) are Playwright **`webServer`** entries — Playwright owns readiness + teardown;
> global-setup only prepares state. No hand-rolled background-process/teardown glue.

`packages/ui/.storybook/visual-regression.spec.ts` (iterate `storybook-static/index.json`):
```ts
import fs from "node:fs";
import { test, expect } from "@playwright/test";

const index = JSON.parse(fs.readFileSync("storybook-static/index.json", "utf8"));
const stories = Object.values<{ id: string; type?: string }>(index.entries).filter(
  (e) => e.type === "story",
);

for (const story of stories) {
  for (const theme of ["light", "dark"] as const) {
    test(`${story.id} [${theme}]`, async ({ page }) => {
      // Multiple globals are separated with `;`, NOT `,` — e.g.
      // `globals=theme:dark;brand:demo`. The comma form silently applies NEITHER global.
      await page.goto(`/iframe.html?id=${story.id}&globals=theme:${theme}`);
      await page.waitForSelector("#storybook-root");
      await expect(page).toHaveScreenshot(`${story.id}--${theme}.png`);
    });
  }
}
```

> **Baseline platform note:** screenshot baselines are platform-sensitive (font rendering).
> If baselines were committed from another OS (strip the platform suffix from snapshot names
> to share them), ubuntu CI may still diff — regenerate baselines ON the CI platform when
> wiring real CI (document the choice in `packages/ui/playwright.config.ts`).

`packages/ui/playwright.config.ts` (VR project against the static build):
```ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: ".storybook",
  testMatch: "visual-regression.spec.ts",
  use: { baseURL: "http://localhost:6006" },
  webServer: {
    command: "npx http-server storybook-static -p 6006 -s",
    url: "http://localhost:6006",
    reuseExistingServer: !process.env.CI,
  },
  // committed baselines live next to the spec; update with --update-snapshots
});
```

`app/.maestro/login.yaml` (local-only mobile flow — the button text is **"Sign in"**, per
the Phase 6 login screen; "Log in" taps nothing):
```yaml
appId: com.example.template
---
- launchApp
- tapOn: "Email"
- inputText: "demo@example.com"
- tapOn: "Password"
- inputText: "Passw0rd!"
- tapOn: "Sign in"
- assertVisible: "Home"
```

**Commands**
```bash
pnpm --filter @platform/template-app add -D @playwright/test serve
pnpm --filter @platform/ui add -D @playwright/test http-server
# commit VR baselines (first run authors them). NOTE the script name is `build-storybook`
# (Phase 2's committed name, also baked into packages/ui/CLAUDE.md) — NOT `storybook:build`:
pnpm --filter @platform/ui build-storybook      # → storybook-static/
pnpm --filter @platform/ui exec playwright test --update-snapshots
# run web E2E locally:
pnpm --filter @platform/template-app exec playwright test
# Maestro (local, needs a running simulator/dev build):
maestro test products/_template/app/.maestro/login.yaml
```

**Why** — PHILOSOPHY.md Testing strategy: web E2E (signup → login → items CRUD → realtime) and
visual regression (Playwright screenshots of the static Storybook build, each story ×
{light,dark}, committed baselines) both run **nightly** in `e2e-nightly.yml` (+
`workflow_dispatch`). Maestro is **local-only initially** (CI via EAS Workflows deferred).
VR visits `iframe.html?id=<story>&globals=theme:dark|light` per the Storybook config note.

---

### (f) Workflows — REAL YAML

> All `secrets.*`, repo owners, and app names below are **clearly-marked placeholders**.
> `example` is the org placeholder; swap on real-infra day.

#### `ci.yml`

**Files** — `.github/workflows/ci.yml`

**Contents**
```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0          # turbo --affected needs history for the base diff
      - uses: jdx/mise-action@v4  # installs Node 24 / pnpm 11 / Python 3.13 / uv from mise.toml
                                   # v4 = Node-24 action runtime (Node 20 is EOL on GH runners); commit a mise.lock for locked installs
      - run: pnpm install --frozen-lockfile
      - name: uv sync (affected APIs)
        run: |
          for api in products/*/api; do
            uv sync --frozen --project "$api"
          done
      - name: Lint / typecheck / test / build / openapi (affected only)
        env:
          # Explicit base/head so `--affected` scopes correctly regardless of squash-merge
          # histories: PR base sha on pull_request, the pushed-from sha on push to main.
          TURBO_SCM_BASE: ${{ github.event.pull_request.base.sha || github.event.before }}
          TURBO_SCM_HEAD: ${{ github.sha }}
        run: pnpm turbo run lint typecheck test build openapi --affected
      - name: Typegen drift check
        run: |
          git diff --exit-code products/*/api-client products/*/api/openapi.json
    services:
      postgres:                   # real Postgres for API integration tests
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready" --health-interval 10s
          --health-timeout 5s --health-retries 5
```
> RESOLVED (affected base ref): with `fetch-depth: 0`, Turborepo 2.x auto-detects the base
> (PR base ref on `pull_request`, previous commit on `push` to `main`) — usually correct. To
> be robust against squash-merge histories and shallow edge cases, the step above sets
> `TURBO_SCM_BASE`/`TURBO_SCM_HEAD` explicitly (`github.event.pull_request.base.sha` on PRs,
> `github.event.before` on push). The `uv sync` loop is the documented "uv sync affected
> apis" — a coarse all-APIs sync; a stricter affected filter can be layered later.

**Commands** — `git push origin <branch>` → Actions runs it. Locally:
`pnpm turbo run lint typecheck test build openapi --affected`.

**Why** — PHILOSOPHY.md Workflows: "ci.yml — mise-action → pnpm frozen install → uv sync (affected
apis) → `turbo run lint typecheck test build openapi --affected` → drift check." The drift
check is the contract guard — a stale `openapi.json` or generated client makes `git diff
--exit-code` non-zero and fails CI.

#### `deploy-api.yml`

**Files** — `.github/workflows/deploy-api.yml`

**Contents**
```yaml
name: Deploy API
on:
  push:
    branches: [main]                    # → staging
    tags: ["*-api-v*"]                  # <product>-api-v* → production
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      products: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v6
      - uses: dorny/paths-filter@v4
        id: filter
        with:
          filters: |
            template: ['products/_template/api/**', 'packages/**']
            demo: ['products/demo/api/**', 'packages/**']
  deploy:
    needs: changes
    if: ${{ needs.changes.outputs.products != '[]' }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        product: ${{ fromJSON(needs.changes.outputs.products) }}
    steps:
      - uses: actions/checkout@v6
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - name: Deploy (staging on main, production on tag)
        working-directory: products/${{ matrix.product == 'template' && '_template' || matrix.product }}/api
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}   # PLACEHOLDER secret
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/* ]]; then
            flyctl deploy -c fly.production.toml --remote-only
          else
            flyctl deploy -c fly.staging.toml --remote-only
          fi
```
> Alembic runs as the Fly **release_command** (Key ruling #4) over the direct 5432
> `DATABASE_MIGRATION_URL` — not a CI step. The `paths-filter` includes `packages/**` so a
> shared-package change can trigger an API redeploy if its image embeds shared TS (typically
> it does not — APIs are Python-only — but the filter is conservative).

**Commands** — staging is automatic on merge to `main`. Production:
`git tag template-api-v1.2.0 && git push origin template-api-v1.2.0`.

**Why** — PHILOSOPHY.md: "deploy-api.yml — paths-filter on `products/*/api/** + packages/**` →
matrix `flyctl deploy -c fly.staging.toml`; tags → prod." Trunk-based: `main` → staging,
`<product>-api-v*` tag → that product's production.

#### `eas.json` + `vercel.json` (per product — created HERE, explicitly)

**Files** — `products/_template/eas.json`, `products/_template/vercel.json`

Both are in PHILOSOPHY's tree but **no earlier phase creates them** (Phase 2 built the app
shell without them — a gap the 2026-07-05 run closed here). The EAS workflows below need
`eas.json`'s profiles/channels, and Vercel needs the SPA rewrite:

```jsonc
// products/_template/eas.json — eas-cli >= 16; appVersionSource remote.
// Channel names are EXACTLY `staging` / `production` (the workflows depend on them).
{
  "cli": { "version": ">= 16.0.0", "appVersionSource": "remote" },
  "build": {
    "staging": { "channel": "staging", "distribution": "internal" },
    "production": { "channel": "production", "autoIncrement": true }
  },
  "submit": { "production": {} }
}
```

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```
(The `vercel.json` SPA rewrite is the web mirror of the desktop `app://` SPA fallback —
`web.output: "single"` produces one `index.html` and client-side routing needs every deep
link rewritten to it.)

#### `eas-build.yml`

**Files** — `.github/workflows/eas-build.yml`

**Contents**
```yaml
name: EAS Build
on:
  workflow_dispatch:
    inputs:
      product: { description: "product token (e.g. template, demo)", required: true }
      profile: { description: "EAS build profile", required: true, default: "production" }
  push:
    tags: ["*-app-v*"]                  # <product>-app-v* → store build
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: jdx/mise-action@v4
      - run: pnpm install --frozen-lockfile     # honours pnpm-workspace.yaml `nodeLinker: hoisted` (pnpm 11)
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}        # PLACEHOLDER secret
      - name: Resolve product token (from tag on tag-push)
        id: tag
        if: startsWith(github.ref, 'refs/tags/')
        run: echo "product=${GITHUB_REF_NAME%%-app-v*}" >> "$GITHUB_OUTPUT"
      - name: EAS build
        # dispatch input wins; on a tag push, parse `<product>` from the `<product>-app-v*` tag.
        # `template` maps to the on-disk `_template` dir (the literal product token vs path).
        working-directory: products/${{ (github.event.inputs.product || steps.tag.outputs.product) == 'template' && '_template' || (github.event.inputs.product || steps.tag.outputs.product) }}/app
        run: eas build --non-interactive --profile "${{ github.event.inputs.profile || 'production' }}"
```
> **eas-cli workspace-detection workaround (gotcha):** the build relies on the committed
> hoisted node-linker (`nodeLinker: hoisted` in `pnpm-workspace.yaml` — pnpm 11 relocates this
> out of `.npmrc`; the `.npmrc` form is still honoured but the canonical home is the workspace
> yaml) AND a `"packageManager": "pnpm@11.x"` field in the **root** `package.json` (match the
> exact pnpm version mise pins) — without both, `eas build` misdetects the package manager in
> a pnpm workspace. RESOLVED (tag→product parse): the `tag` step derives `<product>` from the
> `<product>-app-v*` tag via bash parameter expansion `${GITHUB_REF_NAME%%-app-v*}` and
> exposes it as `steps.tag.outputs.product`; the `working-directory` then maps the literal
> `template` token to the on-disk `_template` dir (same expression the matrix workflows use).

**Commands** — manual: Actions → "EAS Build" → run with `product`+`profile`. Store build:
`git tag template-app-v1.0.0 && git push origin template-app-v1.0.0`.

**Why** — PHILOSOPHY.md: "eas-build.yml — dispatch/tag; needs `EXPO_TOKEN`, committed `.npmrc`,
`packageManager` field in root package.json (eas-cli workspace detection workaround). Store
builds only for native changes."

#### `eas-update.yml`

**Files** — `.github/workflows/eas-update.yml`

**Contents**
```yaml
name: EAS Update (OTA)
on:
  push:
    branches: [main]                    # JS-only OTA → staging channel
    tags: ["*-ota-v*"]                  # <product>-ota-v* → production channel
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      products: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v6
      - uses: dorny/paths-filter@v4
        id: filter
        with:
          filters: |
            template: ['products/_template/app/**', 'packages/**']
            demo: ['products/demo/app/**', 'packages/**']
  update:
    needs: changes
    if: ${{ needs.changes.outputs.products != '[]' }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        product: ${{ fromJSON(needs.changes.outputs.products) }}
    steps:
      - uses: actions/checkout@v6
      - uses: jdx/mise-action@v4
      - run: pnpm install --frozen-lockfile
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          # NOTE: an ${{ }} expression inside a YAML FLOW mapping (`with: { token: ${{...}} }`)
          # is INVALID YAML — use block mapping (or quote the expression).
          token: ${{ secrets.EXPO_TOKEN }}
      - name: OTA (staging on main, production on tag)
        working-directory: products/${{ matrix.product == 'template' && '_template' || matrix.product }}/app
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/* ]]; then
            eas update --channel production --non-interactive --auto
          else
            eas update --channel staging --non-interactive --auto
          fi
```
> **OTA delivery prerequisite (cross-reference Phase 2 `app.config.ts`):** `eas update
> --channel` only reaches INSTALLED builds if `app.config.ts` sets `updates.url`
> (`https://u.expo.dev/<projectId>`) AND a `runtimeVersion` policy (e.g. `{ policy:
> "appVersion" }` or `"fingerprint"`). `extra.eas.projectId` alone does NOT deliver OTA —
> without `updates.url` + a matching `runtimeVersion`, this workflow publishes an update that
> no installed build ever fetches. `eas update:configure` populates both.

**Commands** — staging OTA is automatic on merge to `main`. Production OTA:
`git tag template-ota-v1.0.1 && git push origin template-ota-v1.0.1`.

**Why** — PHILOSOPHY.md: "eas-update.yml — OTA: on main push affecting a product's app → `eas
update --channel staging`; tag `<product>-ota-v*` → `--channel production`." Channels are
EXACTLY `staging` / `production`. Mobile = OTA for JS-only changes; native changes go through
`eas-build.yml`.

#### `e2e-nightly.yml`

**Files** — `.github/workflows/e2e-nightly.yml`

**Contents**
```yaml
name: E2E Nightly
on:
  schedule:
    - cron: "0 4 * * *"          # nightly 04:00 UTC
  workflow_dispatch: {}          # on-demand (the Verify path)
jobs:
  web-e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: jdx/mise-action@v4
      # The web E2E drives the REAL local stack — it needs the Supabase CLI (the harness
      # health-checks kong + uses the local DB), the api's Python env, and the api env vars
      # (a clean CI checkout has NO api/.env):
      - uses: supabase/setup-cli@v1
      - run: supabase start
        working-directory: products/_template
      - run: pnpm install --frozen-lockfile
      - run: uv sync --frozen --project products/_template/api
      - run: pnpm exec playwright install --with-deps chromium
      - name: Web E2E (signup → login → CRUD → realtime)
        env:
          DATABASE_URL: postgresql+psycopg://postgres:postgres@localhost:54322/postgres
          DATABASE_MIGRATION_URL: postgresql+psycopg://postgres:postgres@localhost:54322/postgres
          SUPABASE_URL: http://localhost:54321
          SUPABASE_SERVICE_ROLE_KEY: ${{ secrets.SUPABASE_LOCAL_SERVICE_ROLE_KEY }} # or parse from `supabase status`
        run: pnpm --filter @platform/template-app exec playwright test
  visual-regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: jdx/mise-action@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium
      - name: Build Storybook
        # script name is `build-storybook` (Phase 2's committed name) — NOT `storybook:build`
        run: pnpm --filter @platform/ui build-storybook
      - name: Visual regression (each story × light/dark vs committed baselines)
        run: pnpm --filter @platform/ui exec playwright test
```
> Two independent jobs so a VR diff doesn't mask an E2E failure (and vice versa). VR baselines
> are committed; a diff fails the job and uploads the comparison (add an
> `actions/upload-artifact` step for the Playwright report when wiring CI for real). The
> web-e2e job runs against the SUPABASE LOCAL stack (not a bare postgres service container) —
> the E2E exercises auth + realtime, which need the full stack; baselines may need
> regeneration on ubuntu if they were committed from another OS (see step e).

**Commands** — Actions → "E2E Nightly" → **Run workflow** (`workflow_dispatch`).

**Why** — PHILOSOPHY.md: "e2e-nightly.yml — Playwright E2E + Storybook visual regression
(schedule)" and Testing strategy marks both **nightly + `workflow_dispatch`**.

#### `electron-release.yml`

**Files** — `.github/workflows/electron-release.yml`

**Contents**
```yaml
name: Electron Release
on:
  push:
    tags: ["*-desktop-v*"]       # <product>-desktop-v* → 3-OS matrix
jobs:
  release:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    env:
      # MUST be defined at JOB level: a step-level `env:` is NOT visible inside another
      # step's `if:` expression — with it only step-level, `env.MAC_CSC_LINK` evaluates as
      # empty context in the `if:` and actionlint flags it.
      MAC_CSC_LINK: ${{ secrets.MAC_CSC_LINK }}          # PLACEHOLDER (empty until certs exist)
    steps:
      - uses: actions/checkout@v6
      - uses: jdx/mise-action@v4
      - run: pnpm install --frozen-lockfile
      - name: Resolve product token (from the desktop tag)
        id: tag
        # `shell: bash` is REQUIRED — the windows runner defaults to pwsh, where the bash
        # parameter expansion below is a syntax error.
        shell: bash
        run: echo "product=${GITHUB_REF_NAME%%-desktop-v*}" >> "$GITHUB_OUTPUT"
      - name: Build the web bundle the desktop wraps
        run: pnpm turbo run export:web --filter=*-app
      # macOS: sign+publish only when certs are present; empty placeholders → build-but-don't-publish.
      - name: electron-builder — signed publish (mac when certs exist; win/linux always)
        if: runner.os != 'macOS' || env.MAC_CSC_LINK != ''
        working-directory: products/${{ steps.tag.outputs.product == 'template' && '_template' || steps.tag.outputs.product }}/desktop
        env:
          GH_TOKEN: ${{ secrets.DESKTOP_RELEASES_TOKEN }}   # PLACEHOLDER (token for <org>/<product>-desktop-releases)
          CSC_LINK: ${{ secrets.MAC_CSC_LINK }}              # electron-builder reads CSC_LINK
          CSC_KEY_PASSWORD: ${{ secrets.MAC_CSC_KEY_PASSWORD }}
        run: pnpm electron-builder --publish always
      - name: electron-builder — macOS build-only (no certs yet)
        if: runner.os == 'macOS' && env.MAC_CSC_LINK == ''
        working-directory: products/${{ steps.tag.outputs.product == 'template' && '_template' || steps.tag.outputs.product }}/desktop
        run: pnpm electron-builder --mac --publish never   # builds unsigned; does NOT publish the mac artifact
```
> The tag must match `desktop/package.json` `version`. RESOLVED (tag→product parse): the `tag`
> step derives `<product>` from `${GITHUB_REF_NAME%%-desktop-v*}` (bash parameter expansion),
> mapping the literal `template` token to the on-disk `_template` dir. RESOLVED (macOS
> signing): the publish step is gated `if: runner.os != 'macOS' || env.MAC_CSC_LINK != ''` so
> win/linux always publish and macOS only publishes once `MAC_CSC_LINK`/`MAC_CSC_KEY_PASSWORD`
> certs exist; until then the macOS leg runs a `--publish never` build-only step (unsigned, no
> publish). Alternatively drop `macos-latest` from the matrix entirely. macOS auto-update
> requires a signed AND notarized app, so mac OTA stays inert until certs land.

**Commands** — `git tag template-desktop-v1.0.0 && git push origin template-desktop-v1.0.0`.

**Why** — PHILOSOPHY.md: "electron-release.yml — 3-OS matrix, `electron-builder --publish always`,
tag must match `desktop/package.json` version." Each product's desktop publishes to its own
`<org>/<product>-desktop-releases` repo (Key ruling #3 — avoids the electron-updater
"latest release of the repo" collision).

> **Note — web has NO workflow.** Vercel's git integration deploys each product's web app on
> push. There is deliberately no `web-deploy.yml`. For skipping unaffected monorepo builds,
> **`turbo-ignore` is now OPTIONAL**: Vercel ships a built-in **"Automatically skip
> unnecessary deployments in monorepos"** project setting (Turborepo-powered) that skips
> unchanged projects with NO manual ignored-build-step config — prefer enabling that. The
> per-product `npx turbo-ignore` "ignored build step" remains a valid manual alternative; if
> still invoked bare, pass **`--fallback=HEAD^`** (`npx turbo-ignore --fallback=HEAD^`) to
> avoid the new-branch "always deploys" gotcha (turbo-ignore otherwise compares against the
> last successful deployment on the branch, which doesn't exist on a branch's first commit).

---

### (g) Docs & agent surface — root + packages/ui + product

> PHILOSOPHY.md "Docs & agent surface": README + CLAUDE.md + `.claude/commands/` at THREE levels.
> Product-level docs are authored once in `_template` and **token-rewritten by the generator**
> (`new-product.mjs` step 3 covers product README/CLAUDE.md/`.claude/commands/*`; ports and
> infra names come from `product.json`).

#### Root

**Files** — `README.md`, `CLAUDE.md`, `.claude/commands/{new-product,affected,typegen,release,add-component,sync-tokens,bootstrap-design-system}.md`

**Contents** — root `CLAUDE.md` is the monorepo map + conventions + gotchas:
```markdown
# CLAUDE.md — platform monorepo

## Map
packages/{config,ui,core}; products/{_template,demo}/{app,desktop,api,api-client}.

## Conventions (locked)
- Promote-on-2nd-use: compositions start product-local; move into packages/* on 2nd use.
- Naming derives from the PRODUCT, never the repo: @platform/*, com.example.*,
  infra <org>-<product>-<env> (org placeholder `example`).
- Theming = semantic CSS variables. NEVER name a color in a component — tokens only.
- Figma modes ARE brand modes; theme.ts is the export of a Figma brand mode.
- Realtime is BROADCAST-ONLY. No Postgres-Changes, no RLS holes.
- Errors are RFC 9457 problem+json; the generated api-client is NEVER hand-edited.

## Gotchas
- pnpm hoisted linker (`nodeLinker: hoisted` in pnpm-workspace.yaml, pnpm 11); never set disableHierarchicalLookups.
- Supabase pooler 6543 = transaction-mode only (psycopg3, NullPool, prepare_threshold=None);
  Alembic migrates over direct 5432 (DATABASE_MIGRATION_URL).
- Sentry = @sentry/react-native (NOT sentry-expo).
- X-Request-Id: client → API → logs; same id tags Sentry on both sides.

## Commands
/new-product <name> · /affected · /typegen <product> · /release <product> <surface>
/add-component <name> · /sync-tokens · /bootstrap-design-system
```
Root `.claude/commands/` inventory (each a thin runnable recipe): `new-product.md`
(`node scripts/new-product.mjs $ARG`), `affected.md` (`turbo run lint typecheck test build
--affected`), `typegen.md` (`turbo run openapi build --filter=*$ARG-api-client`),
`release.md` (tag `<product>-<surface>-v*`, push), `add-component.md` (delegates to the
`packages/ui` recipe), `sync-tokens.md` (`node scripts/figma-tokens.mjs`),
`bootstrap-design-system.md` (the handover procedure: reconcile → tokens → components → verify).

> `/add-component`, `/sync-tokens`, `/bootstrap-design-system` operate on shared `packages/ui`
> (no product arg); the others take a product arg (PHILOSOPHY.md Docs & agent surface).

**The `ptfm-*` agentic development pipeline (also root `.claude/commands/`).** Alongside the
thin wrappers above, the root command surface includes the opinionated lifecycle pipeline that
is the **primary way products are built**: `ptfm-product` → `ptfm-architect` → `ptfm-plan` →
`ptfm-implement` → `ptfm-audit` → `ptfm-simplify` → `ptfm-commonify` → `ptfm-review` →
`ptfm-test-ui`. Each takes the **product name as its first arg** and writes its artifact to that
product's own docs tree: `products/<product>/docs/{product,architecture,plans,implementation,
reviews}/` (created on first write — no pre-seeding; the *product* already exists via
`new-product`). These commands encode PHILOSOPHY.md's invariants as executable flows; they are
already authored at `.claude/commands/ptfm-*.md` and **must NOT be deleted by the post-setup
cleanup** (they are runtime, not build scaffolding).

**Step — wire the agentic tooling (do this as part of the agent-surface build).** For the
`ptfm-*` pipeline to work, the implementing agent must:
1. Keep the `ptfm-*.md` command files in `.claude/commands/` (they ship with the repo).
2. Document the **operational stack** (the MCP integrations the pipeline drives) in the root
   `README.md` and root `CLAUDE.md`: **Linear** (`mcp__Linear__*`), **Notion**
   (`mcp__Notion__*`), **Figma** (`mcp__Figma__*`), **Supabase** (`mcp__Supabase__*`,
   read-only introspection — migrations go via Alembic), **Playwright** (`mcp__playwright__*`),
   **GitHub** (`mcp__github__*`) — and that a developer must connect them in Claude Code before
   running the pipeline.
3. State in each product's `CLAUDE.md` that the per-product `docs/{product,architecture,plans,
   implementation,reviews}/` tree is where the pipeline's artifacts live, and that the pipeline
   is the canonical build workflow for that product.

Root `README.md` = human quickstart: `mise install && pnpm install && pnpm bootstrap`; where
components live; `pnpm --filter @platform/ui storybook`; `pnpm new-product <name>`; points at
`CLAUDE.md` for authoritative recipes (does not duplicate them). The README at this stage is
still build-oriented (Status / Stage 1); **Phase 9 (`/implement 9`) rewrites it into its
built-state form** and strips the build scaffolding — do not hand-write a manual "post-setup
cleanup" section here, that step is automated by Phase 9.

#### packages/ui

**Files** — `packages/ui/CLAUDE.md`, `packages/ui/FIGMA.md`,
`packages/ui/.claude/commands/{add-component,sync-tokens,bootstrap-design-system}.md`

**Contents** — `packages/ui/CLAUDE.md` is the design-system runbook (symmetric to the api
CLAUDE.md). The **add-a-component recipe** (enforced verbatim):
```markdown
# CLAUDE.md — @platform/ui design system

## add-a-component recipe (run via /add-component)
1. cli-add (react-native-reusables CLI) OR author the component into
   src/components/ui/<name>.tsx — OWNED source (shadcn model), tokens-only.
2. Pin @rn-primitives/* deps EXACT (pre-1.0).
3. Write <name>.stories.tsx — ONE story per cva variant.
4. Write <name>.figma.tsx — Code Connect map (Figma props → cva variants).
5. Export from src/index.ts.
6. Commit the VR baseline (light + dark) — see e2e-nightly VR.

## Invariants
- Tokens ONLY. NEVER name a color (no hex, no brand values) — use bg-primary etc.
- Two-tier ownership: tier-1 owned primitives here; tier-2 compositions start
  product-local, promote here on 2nd use.
- theme.ts / global.css default light+dark are generated by /sync-tokens
  (figma-tokens.mjs) — NEVER hand-edit generated theme values.

## Storybook
pnpm --filter @platform/ui storybook — toolbar has light/dark + brand (template/demo).

## Figma
Code Connect maps are authored as *.figma.tsx and published via the Code Connect CLI.
See FIGMA.md for the designer-side library conventions. /bootstrap-design-system is the
handover-day import (reconcile → tokens → components → verify).
```
`packages/ui/FIGMA.md` (designer-facing — the single doc handed to design): Variables
structure (`primitives` raw scale + `semantic` collection), **modes = light/dark × brand
(template/demo)**, names-as-API, component anatomy must match the code, publish as a team
library (Foundations = Variables, Components = component sets). `.claude/commands/`:
`add-component.md` (the recipe above), `sync-tokens.md` (`node scripts/figma-tokens.mjs`),
`bootstrap-design-system.md`.

#### Product (`_template`, token-rewritten by the generator)

**Files** — `products/_template/README.md`, `products/_template/CLAUDE.md`,
`products/_template/api/.../CLAUDE.md` (nested api recipe),
`products/_template/.claude/commands/{dev,typegen,migrate,add-feature,release}.md`

**Contents** — product `CLAUDE.md`: product structure, **ports + infra names** (sourced from
`product.json` so they stay accurate after stamping), where compositions live
(`features/<x>/components/`) + the promote-on-2nd-use trigger, and that the product's
`theme.ts` is the export of its Figma brand mode. The **nested api `CLAUDE.md`** holds the
**add-an-endpoint-end-to-end recipe** (enforced verbatim):
```markdown
# CLAUDE.md — template api

## add-an-endpoint recipe
model (SQLModel, UUIDv7 base, RLS deny-all migration)
  → service (class per aggregate, holds the session via Depends, owns logic + data access)
  → schema (Pydantic v2 DTO — the ONLY thing crossing HTTP; ORM models never serialized)
  → router (thin; depends on the service; maps schema↔domain)
  → openapi (turbo run openapi) → typegen (turbo run build --filter=*api-client)
  → hook (generated TanStack hook) → screen (features/<x>)

## Rules
- Strict layered OOP, NO repository layer (services query directly).
- pyright strict + Pydantic strict — enforced in pre-push AND CI.
- RFC 9457 problem+json errors; cursor pagination (useInfiniteQuery-ready).
```
Product `.claude/commands/` (product-scoped, apply when a session opens in the product dir):
`dev.md` (`turbo run dev --filter=*<product>-*`), `typegen.md` (`turbo run openapi build
--filter=*<product>-api-client`), `migrate.md` (`uv run alembic …`), `add-feature.md`
(scaffold `features/<x>/`), `release.md` (tag `<product>-<surface>-v*`).
Product `README.md` = human quickstart (where components live, launch the workbench, sync
tokens) pointing at the CLAUDE.md for the recipe.

**Commands**
```bash
# verify the agent surface exists at all three levels:
ls CLAUDE.md README.md .claude/commands/
ls packages/ui/CLAUDE.md packages/ui/FIGMA.md packages/ui/.claude/commands/
ls products/_template/CLAUDE.md products/_template/README.md products/_template/.claude/commands/
```

**Why** — PHILOSOPHY.md Docs & agent surface (the long bullet + ruling): three-level docs, the
add-a-component recipe in `packages/ui/CLAUDE.md`, the add-an-endpoint recipe in the nested
api CLAUDE.md, FIGMA.md as the designer doc, root commands take a product arg except
`/add-component` `/sync-tokens` `/bootstrap-design-system`, product commands are
product-scoped and load from the session's project root.

---

## Gotchas & pitfalls

- **Affected-only caching proof.** `--affected` must rebuild ONLY touched products; touching
  `products/demo` must leave `template` a **cache hit**. This depends on correct turbo
  `inputs`/`outputs` (esp. the mandatory Python `inputs` globs from Phase 1) and on
  `fetch-depth: 0` in CI so Turbo can compute the base diff. If a shared `packages/*` is
  touched, **all dependents rebuild** (the co-evolve guard) — that is correct, not a cache
  miss bug.
- **Drift check is the contract guard.** `turbo run openapi build --filter=*api-client*`
  regenerates `openapi.json` + the client; `git diff --exit-code products/*/api-client
  products/*/api/openapi.json` must be clean. A model change without a regen → non-zero diff
  → CI red. Never hand-edit the generated client.
- **Broadcast-only — never Postgres-Changes.** The realtime path is service-role HTTP
  broadcast on a per-product channel; clients refetch through the API. Do NOT subscribe to
  Postgres-Changes and do NOT open RLS on any table to enable a subscription — tables stay
  deny-all and the schema stays private. Opening RLS "just to get realtime" is the mistake
  this pattern exists to prevent.
- **Expo Go can't receive push tokens.** `getExpoPushTokenAsync()` needs a **dev build**
  (custom dev client), not Expo Go, and a **real device** (not a simulator —
  `Device.isDevice` guards this). The full push loop is therefore **verified later on real
  devices**; CI verifies the server side (`send_push()` with a mocked httpx transport).
- **Sentry SDK is `@sentry/react-native`, NOT `sentry-expo`.** `sentry-expo` is deprecated;
  PHILOSOPHY.md locks `@sentry/react-native`. Using the wrong package is a silent footgun.
- **eas-cli workspace detection workaround.** In a pnpm workspace, `eas build`/`eas update`
  misdetect the package manager unless BOTH the committed hoisted node-linker
  (`nodeLinker: hoisted` in `pnpm-workspace.yaml` — pnpm 11's home for it; the old `.npmrc`
  `node-linker` key is silently ignored on pnpm 11) and a `"packageManager": "pnpm@11.x"` field in the
  **root** `package.json` are present.
  Both must ship.
- **Web has NO workflow.** Do not add a `web-deploy.yml`. Vercel git integration handles web;
  adding a workflow would double-deploy. Skip-unaffected is now Vercel's built-in
  "Automatically skip unnecessary deployments in monorepos" setting (preferred) — `npx
  turbo-ignore` as the manual "ignored build step" is OPTIONAL now, and if invoked bare must
  pass `--fallback=HEAD^` to avoid the new-branch always-deploy gotcha.
- **macOS desktop signing gating.** `electron-builder --publish always` only signs/notarizes
  macOS once certs exist. The workflow gates the publish step `if: runner.os != 'macOS' ||
  env.MAC_CSC_LINK != ''` (win/linux always publish; macOS publishes only when the cert
  secret is set) and falls back to an unsigned `--mac --publish never` build-only step when
  certs are absent. Alternatively drop `macos-latest` from the matrix. Auto-update on macOS
  requires signing AND notarization (PHILOSOPHY.md Electron essentials).
- **Request-id middleware ordering.** Register `RequestIdMiddleware` LAST (`add_middleware`
  adds outermost-last) so it wraps the security middleware and error handlers — otherwise
  error responses won't carry the `X-Request-Id` header and error logs lose the id.
- **All repo/org values are placeholders.** `example`, `com.example.*`, `TODO-EAS-PROJECT-ID`,
  `<org>/<product>-desktop-releases`, every `secrets.*`, and Fly app names
  (`example-template-api-stg|prod`) are clearly-marked swap-points for real-infra day. A
  `git grep -inE 'example|TODO'` should surface exactly these and nothing else. (The former
  `PARSE-FROM-TAG` placeholders in `eas-build.yml`/`electron-release.yml` are now resolved to
  real `${GITHUB_REF_NAME%%-<surface>-v*}` parse steps.)
- **Port-squatting when re-verifying servers.** A leftover E2E `uvicorn` can squat port 8000
  and silently serve a SECOND instance's curl checks — before re-verifying, find the PID via
  `netstat` and kill it (on Windows, kill `electron.exe`/`python.exe` itself, not a shim).
- **Shared-core skeletons must stay product-agnostic.** Any `core/*` snippet that names
  `@platform/<product>-api-client` is a bug by construction — core is never stamped. The
  injected forms in steps (a)/(b)/(c) (`configureApiClient(client, …)`,
  `registerForPushNotifications(post)`, the `keys` map) are the pattern; a product passes
  its own generated symbols from its own `_layout.tsx`/feature code.

---

## Verification

Maps 1:1 to the Phase 8 Verify row.

1. **Push branch → CI green.**
   ```bash
   git switch -c phase-8-cicd
   git push -u origin phase-8-cicd
   # GitHub → Actions → "CI" run is green (lint/typecheck/test/build/openapi + drift)
   ```
   > Caveat: `ci.yml` triggers on `pull_request` + push to `main` only — a bare
   > feature-branch push does NOT fire it. Open a PR, or (if PRs aren't possible in the
   > session) run the CI steps locally as the evidence:
   > `pnpm turbo run lint typecheck test build openapi --affected` + the drift check.
2. **Touch one product → other is cache-hit.**
   ```bash
   # touch demo only:
   echo "" >> products/demo/api/src/demo_api/main.py
   pnpm turbo run build --affected
   # turbo summary: demo tasks EXECUTED, template tasks "cache hit, replaying logs"
   ```
3. **Stale `openapi.json` fails drift check.**
   ```bash
   # change a response model WITHOUT regenerating:
   # (edit a schemas/*.py field) then:
   git diff --exit-code products/*/api-client products/*/api/openapi.json   # → non-zero (fails)
   # fix: turbo run openapi build --filter=*template-api-client && git add -A
   ```
4. **Items list refreshes across two open clients after a mutation.**
   ```bash
   pnpm bootstrap                                  # supabase local + API + app
   pnpm --filter @platform/template-app exec playwright test e2e/items.spec.ts
   # the "broadcast item" assertion on the second context passing IS this proof.
   # Manual: open localhost:8081 in two tabs, add an item in one → the other refreshes.
   ```
   > The STAMPED product's harness is proof the port-derivation holds: in the 2026-07-05
   > audit, `products/demo/app`'s E2E ran end-to-end on demo's OWN stack (kong 54421, DB
   > 54422, API 8010, `demo:realtime` channel, demo's service-role key) — 1 passed. Run it
   > after any generator change.
5. **API log lines carry the `request_id`.**
   ```bash
   curl -s -H "X-Request-Id: test-rid-123" http://localhost:8000/v1/hello
   # API stdout shows a JSON line: {"event":"http_request",...,"request_id":"test-rid-123"}
   # and the response carries `X-Request-Id: test-rid-123` (curl -i to see it).
   ```
6. **`e2e-nightly.yml` green via `workflow_dispatch` (E2E + visual regression).**
   ```bash
   # GitHub → Actions → "E2E Nightly" → Run workflow (branch: main)
   # both jobs (web-e2e, visual-regression) green.
   # Locally: pnpm --filter @platform/ui build-storybook &&
   #          pnpm --filter @platform/ui exec playwright test   # VR vs baselines
   ```
7. **Scheduled task runs via `fly machine run`.**
   ```bash
   fly machine run --app example-template-api-stg \
     registry.fly.io/example-template-api-stg:latest \
     python -m template_api.tasks prune-push-tokens
   # Fly logs show the JSON line {"event":"pruned_push_tokens","count":N}
   ```
   > While the Fly app is still a placeholder (no real infra), the equivalent local
   > evidence: run the task module directly against the real local DB —
   > `uv run python -m template_api.tasks prune-push-tokens` emits the exact documented
   > JSON line; a bare invocation prints usage and exits 2.

---

## Commits

Logical commits on the `phase-8-cicd` branch (PHILOSOPHY.md: each phase = one or a few logical
commits):

1. `feat(obs): request_id middleware + structlog JSON + Sentry both sides + X-Request-Id` —
   step (a).
2. `feat(push): push tests (mocked httpx) + injected core registration + app wiring` —
   step (b) (the model/router/service/migration shipped in Phase 3 — verified here).
3. `feat(realtime): broadcast-only invalidation (api broadcast + core subscribe-and-invalidate)` —
   step (c).
4. `feat(api): scheduled tasks.py prune-push-tokens + Fly machine docs` — step (d).
5. `test(e2e): Playwright web E2E + Storybook VR baselines + Maestro flow` — step (e).
6. `ci: ci/deploy-api/eas-build/eas-update/e2e-nightly/electron-release workflows` — step (f).
7. `docs: root + packages/ui + product CLAUDE.md/README/.claude commands` — step (g).

(Commit/branch/push only when the user asks; do not run git as part of writing this guide.)

---

## Open questions / deferred

- ⚠️ **OPEN / TO CONFIRM — broadcast failure policy:** whether a Supabase broadcast failure
  should fail the mutation. Default here: log + swallow (writes never blocked by a Realtime
  outage). Confirm per product.
- ⚠️ **OPEN / TO CONFIRM — "stale" push-token definition:** PHILOSOPHY.md says "prune stale push
  tokens" without a threshold; the gospel is silent. **90 days by `updated_at` is now the
  ALIGNED default across Phase 3 (`prune_stale(older_than_days=90)`) and this phase** —
  tune per product. (`updated_at` already exists on Phase 3's `UUIDModel` base.)
- **RESOLVED — E2E process orchestration:** Playwright **multi-`webServer`** owns both
  long-lived processes (`serve -s dist`, uvicorn) — readiness + teardown included;
  global-setup only prepares state (supabase up-check, migrate, seed, export). No
  hand-rolled background/teardown glue.
- **RESOLVED — `--affected` base ref in CI:** with `fetch-depth: 0` Turbo auto-detects the
  base (PR base ref / previous push commit); `ci.yml` also sets `TURBO_SCM_BASE`/`TURBO_SCM_HEAD`
  explicitly to be robust against squash-merge histories. Revisit only if CI mis-scopes.
- **RESOLVED — tag→product parsing:** `eas-build.yml` and `electron-release.yml` now derive
  `<product>` from the tag via a step (`echo "product=${GITHUB_REF_NAME%%-<surface>-v*}" >>
  "$GITHUB_OUTPUT"`) and map the literal `template` token to the `_template` dir — no more
  `PARSE-FROM-TAG` placeholder.
- **RESOLVED — macOS signing:** `electron-release.yml` gates the signed publish step with
  `if: runner.os != 'macOS' || env.MAC_CSC_LINK != ''` (win/linux always publish; macOS only
  when certs exist) and runs an unsigned `--publish never` build-only step otherwise. mac
  auto-update stays inert until the app is signed AND notarized.
- **Deferred (PHILOSOPHY.md):** Maestro CI via EAS Workflows (local-only for now); Chromatic
  (declined — self-hosted Playwright VR); ADR-vs-ARCHITECTURE.md decision-record format.
