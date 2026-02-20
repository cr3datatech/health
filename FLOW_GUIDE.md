# Flow Guide: User Page Load → LLM Response Display

This document traces the complete path from when a user opens the page to where the LLM response is displayed on screen.

---

## Architecture Overview

| Layer | Technology | File |
|-------|-----------|------|
| Frontend framework | Next.js 16 (React 19) | `pages/` |
| Authentication | Clerk (`@clerk/nextjs`) | `pages/_app.tsx` |
| Landing page | React (client component) | `pages/index.tsx` |
| Protected app page | React (client component) | `pages/product.tsx` |
| Backend API | FastAPI (Python) | `api/index.py` |
| Auth middleware | `fastapi-clerk-auth` | `api/index.py` |
| Streaming transport | Server-Sent Events (SSE) | `api/index.py` → `pages/product.tsx` |
| LLM | OpenAI API | `api/index.py` |
| Markdown rendering | `react-markdown` + `remark-gfm` | `pages/product.tsx` |

---

## Pages at a Glance

| URL | File | Access |
|-----|------|--------|
| `/` | `pages/index.tsx` | Public — landing page |
| `/product` | `pages/product.tsx` | Requires sign-in + active subscription |
| `/api` | `api/index.py` | Requires valid Clerk JWT Bearer token |

---

## Step-by-Step Flow

### 1. App Bootstrap — ClerkProvider wraps everything

**File**: `pages/_app.tsx`

```tsx
import { ClerkProvider } from '@clerk/nextjs';

export default function App({ Component, pageProps }: AppProps) {
    return (
        <ClerkProvider>
            <Component {...pageProps} />
        </ClerkProvider>
    );
}
```

- `ClerkProvider` initialises Clerk using `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` from `.env.local`
- All pages now have access to Clerk's auth hooks and components
- Clerk automatically manages session cookies and JWT tokens

---

### 2. User Opens the Landing Page (`/`)

**File**: `pages/index.tsx`

- Next.js renders the `Home` component
- Clerk's `<SignedOut>` / `<SignedIn>` components inspect the session:
  - **Not signed in** → shows Sign In button (`clerk.openSignIn()` modal)
  - **Signed in** → shows "Go to App" link and `<UserButton>`

```tsx
<SignedOut>
    <SignInButton mode="modal">
        <button>Sign In</button>
    </SignInButton>
</SignedOut>
<SignedIn>
    <Link href="/product">Go to App</Link>
    <UserButton showName={true} />
</SignedIn>
```

---

### 3. User Signs In

- Clerk opens its hosted sign-in modal
- On success, Clerk sets a session cookie and issues a JWT
- The JWT contains standard claims plus Clerk-specific ones:
  - `sub` — user ID
  - `pla` — subscription plan (e.g. `u:premium_subscription`)
- `<SignedIn>` re-renders to show the "Go to App" button

---

### 4. User Navigates to `/product`

**File**: `pages/product.tsx`

The `Product` component renders immediately and wraps `IdeaGenerator` inside Clerk's `<Protect>` component:

```tsx
<Protect
    condition={(has) =>
        has({ plan: 'pro_plan' }) || has({ plan: 'premium_subscription' })
    }
    fallback={<PricingTable />}
>
    <IdeaGenerator />
</Protect>
```

- Clerk evaluates the session's subscription claims **client-side**
- **No active plan** → renders `<PricingTable />` (Clerk's hosted billing UI)
- **Has `pro_plan` or `premium_subscription`** → renders `<IdeaGenerator />`

---

### 5. IdeaGenerator Mounts and Requests a JWT

**File**: `pages/product.tsx` — `IdeaGenerator` component

```tsx
const { getToken } = useAuth();

useEffect(() => {
    (async () => {
        const jwt = await getToken();
        // ...
    })();
}, []);
```

- `getToken()` returns the current Clerk session JWT
- This token will be sent as `Authorization: Bearer <jwt>` to the backend

---

### 6. Frontend Opens SSE Connection with Auth Header

**File**: `pages/product.tsx`

```tsx
await fetchEventSource('/api', {
    headers: { Authorization: `Bearer ${jwt}` },
    onmessage(ev) {
        buffer += ev.data;
        setIdea(buffer);
    },
    onerror(err) {
        throw err; // Stop retrying on fatal errors
    }
});
```

- Uses `@microsoft/fetch-event-source` instead of native `EventSource`
  - Native `EventSource` **cannot** set custom headers
  - `fetchEventSource` supports custom headers and better error handling
- Opens a persistent HTTP connection to `/api`
- The `Authorization` header carries the Clerk JWT

---

### 7. FastAPI Receives the Request and Verifies the JWT

**File**: `api/index.py`

```python
clerk_config = ClerkConfig(jwks_url=os.getenv("CLERK_JWKS_URL"))
clerk_guard  = ClerkHTTPBearer(clerk_config)

@app.get("/api")
def idea(creds: HTTPAuthorizationCredentials = Depends(clerk_guard)):
```

- `fastapi-clerk-auth` extracts the Bearer token from the `Authorization` header
- Fetches Clerk's public JWKS from `CLERK_JWKS_URL` and verifies the JWT signature
- Rejects the request with 401 if invalid/expired
- On success, decoded JWT claims are available via `creds.decoded`

---

### 8. Backend Reads Plan and Selects Model

**File**: `api/index.py`

```python
pla = creds.decoded.get("pla", "")
subscription_plan = pla.removeprefix("u:").removeprefix("o:") if pla else "free"

if subscription_plan == "premium_subscription":
    model = "gpt-5.1"
elif subscription_plan == "pro_plan":
    model = "gpt-5-nano"
else:
    model = "gpt-5-nano"
```

- The `pla` claim contains the plan slug prefixed with `u:` (user) or `o:` (org)
- The prefix is stripped to get the plain slug
- Model is assigned by tier: `premium_subscription` → `gpt-5.1`, others → `gpt-5-nano`

---

### 9. Backend Calls OpenAI with Streaming

**File**: `api/index.py`

```python
client = OpenAI()
stream = client.chat.completions.create(model=model, messages=prompt, stream=True)
```

- OpenAI SDK streams the response as a generator of delta chunks
- Each chunk contains a small piece of the final text

---

### 10. Backend Streams SSE Response

**File**: `api/index.py`

```python
def event_stream():
    yield (
        f"data: *Model: {model}, Subscription Plan: {subscription_plan}*\n"
        "data: \n"
        "data: ---\n"
        "data: \n"
        "data: **JWT Claims:**\n"
        f"{jwt_claim_lines}\n"
        "data: \n"
        "data: ---\n"
        "data: \n\n"
    )
    for chunk in stream:
        text = chunk.choices[0].delta.content
        if text:
            # Each line must be a separate `data:` event for correct SSE parsing
            lines = text.split("\n")
            for line in lines[:-1]:
                yield f"data: {line}\n\n"
                yield "data:  \n"
            yield f"data: {lines[-1]}\n\n"

return StreamingResponse(event_stream(), media_type="text/event-stream")
```

- First yields a header block: model name, plan, and all JWT claims
- Then streams each OpenAI delta chunk formatted as SSE (`data: <text>\n\n`)
- Multi-line chunks are split so each line is its own SSE event

---

### 11. Frontend Accumulates Chunks and Re-renders

**File**: `pages/product.tsx`

```tsx
onmessage(ev) {
    buffer += ev.data;
    setIdea(buffer);
}
```

- Each SSE message appends to `buffer`
- `setIdea(buffer)` triggers a React re-render on every chunk — streaming effect

---

### 12. ReactMarkdown Renders the Content

**File**: `pages/product.tsx`

```tsx
<ReactMarkdown remarkPlugins={[remarkGfm, remarkBreaks]}>
    {idea}
</ReactMarkdown>
```

- `ReactMarkdown` converts accumulated markdown to HTML on every re-render
- `remarkGfm` — GitHub Flavored Markdown (tables, strikethrough, etc.)
- `remarkBreaks` — converts single newlines to `<br>` tags
- Content appears incrementally as chunks arrive

---

## Data Flow Summary

```
User opens /
    ↓
ClerkProvider initialises (publishable key from .env.local)
    ↓
Landing page: <SignedOut> shows Sign In / <SignedIn> shows Go to App
    ↓
User signs in → Clerk issues JWT with `pla` claim
    ↓
User navigates to /product
    ↓
<Protect> checks plan claim → shows <PricingTable> or <IdeaGenerator>
    ↓
IdeaGenerator mounts → getToken() fetches JWT
    ↓
fetchEventSource('/api', { Authorization: Bearer <jwt> })
    ↓
FastAPI verifies JWT via Clerk JWKS
    ↓
Reads pla claim → selects model (gpt-5.1 or gpt-5-nano)
    ↓
OpenAI streams delta chunks → FastAPI formats as SSE
    ↓
Frontend buffer accumulates chunks → setIdea() → re-render
    ↓
ReactMarkdown renders markdown → content appears incrementally
```

---

## Installed Packages

### Frontend (Node.js) — `package.json`

```bash
npm install
```

| Package | Version | Purpose |
|---------|---------|---------|
| `next` | 16.1.1 | React framework, routing, dev server |
| `react` | 19.2.3 | UI component library |
| `react-dom` | 19.2.3 | React DOM renderer |
| `@clerk/nextjs` | ^6 | Clerk auth: `ClerkProvider`, `<Protect>`, `<PricingTable>`, `useAuth`, `getToken` |
| `@microsoft/fetch-event-source` | ^2.0.1 | SSE client supporting custom headers (needed for Authorization) |
| `react-markdown` | ^10.1.0 | Renders markdown string as HTML |
| `remark-gfm` | ^4.0.1 | GitHub Flavored Markdown plugin |
| `remark-breaks` | ^4.0.0 | Converts `\n` to `<br>` |
| `@tailwindcss/typography` | ^0.5.19 | Tailwind prose styling |
| `tailwindcss` | ^4 | Utility-first CSS framework |
| `typescript` | ^5 | TypeScript compiler |

### Backend (Python) — `requirements.txt`

```bash
pip install -r requirements.txt
```

| Package | Purpose |
|---------|---------|
| `fastapi` | Web framework — defines the `/api` route |
| `uvicorn` | ASGI server to run FastAPI |
| `openai` | Official OpenAI Python SDK (streaming support) |
| `fastapi-clerk-auth` | FastAPI dependency that verifies Clerk JWTs via JWKS |

### Environment Variables — `.env.local`

| Variable | Used by | Purpose |
|----------|---------|---------|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | Frontend (Clerk) | Initialises `ClerkProvider` |
| `CLERK_SECRET_KEY` | Clerk dashboard sync | Backend Clerk config |
| `CLERK_JWKS_URL` | `fastapi-clerk-auth` | URL to fetch Clerk's public keys for JWT verification |
| `OPENAI_API_KEY` | `OpenAI()` client | Authenticates OpenAI API calls |

---

## Important Files Reference

| File | Role |
|------|------|
| `pages/_app.tsx` | Root — wraps app in `ClerkProvider` |
| `pages/index.tsx` | Landing page — sign-in / pricing preview |
| `pages/product.tsx` | Protected page — plan gate + idea generator + SSE client |
| `api/index.py` | API endpoint — JWT verification, model selection, SSE streaming |
| `.env.local` | Environment variables (not committed) |
| `next.config.ts` | Next.js configuration (API proxy to FastAPI) |
