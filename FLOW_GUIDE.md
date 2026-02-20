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
| LLM | OpenAI API (`gpt-4o-mini`) | `api/index.py` |
| Markdown rendering | `react-markdown` + `remark-gfm` | `pages/product.tsx` |

---

## Pages at a Glance

| URL | File | Access |
|-----|------|--------|
| `/` | `pages/index.tsx` | Public — landing page |
| `/product` | `pages/product.tsx` | Requires sign-in + `premium_subscription` plan |
| `/api` | `api/index.py` | POST — requires valid Clerk JWT Bearer token |

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

- Next.js renders the `Home` component for MediNotes Pro
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
- `<SignedIn>` re-renders to show the "Go to App" / "Open Consultation Assistant" button

---

### 4. User Navigates to `/product`

**File**: `pages/product.tsx`

The `Product` component renders immediately and wraps `ConsultationForm` inside Clerk's `<Protect>` component:

```tsx
<Protect
    plan="premium_subscription"
    fallback={<PricingTable />}
>
    <ConsultationForm />
</Protect>
```

- Clerk evaluates the session's subscription claims **client-side**
- **No `premium_subscription` plan** → renders `<PricingTable />` (Clerk's hosted billing UI)
- **Has `premium_subscription`** → renders `<ConsultationForm />`

---

### 5. ConsultationForm Renders — User Fills in the Form

**File**: `pages/product.tsx` — `ConsultationForm` component

Unlike the saas idea generator (which fires immediately on mount), the health app is **form-driven** — the user fills in three fields and clicks a button:

```tsx
const [patientName, setPatientName] = useState('');
const [visitDate, setVisitDate]     = useState<Date | null>(new Date());
const [notes, setNotes]             = useState('');
const [output, setOutput]           = useState('');
const [loading, setLoading]         = useState(false);
```

- **Patient Name** — text input
- **Date of Visit** — `react-datepicker` date picker (formatted as `yyyy-MM-dd`)
- **Consultation Notes** — multi-line textarea
- **Generate Summary** button triggers `handleSubmit`

---

### 6. User Submits — Frontend Fetches JWT and Opens SSE

**File**: `pages/product.tsx`

```tsx
async function handleSubmit(e: FormEvent) {
    e.preventDefault();
    setOutput('');
    setLoading(true);

    const jwt = await getToken();

    await fetchEventSource('/api', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${jwt}`,
        },
        body: JSON.stringify({
            patient_name: patientName,
            date_of_visit: visitDate?.toISOString().slice(0, 10),
            notes,
        }),
        onmessage(ev) {
            buffer += ev.data;
            setOutput(buffer);
        },
        onclose() { setLoading(false); },
        onerror(err) {
            controller.abort();
            setLoading(false);
        },
    });
}
```

- `getToken()` returns the current Clerk session JWT
- `fetchEventSource` sends a **POST** (not GET) with a JSON body
  - Native `EventSource` cannot POST or set custom headers — `@microsoft/fetch-event-source` is required
- The `Authorization: Bearer <jwt>` header carries authentication
- The body contains `patient_name`, `date_of_visit`, and `notes`

---

### 7. FastAPI Receives the POST and Verifies the JWT

**File**: `api/index.py`

```python
clerk_config = ClerkConfig(jwks_url=os.getenv("CLERK_JWKS_URL"))
clerk_guard  = ClerkHTTPBearer(clerk_config)

@app.post("/api")
def consultation_summary(
    visit: Visit,
    creds: HTTPAuthorizationCredentials = Depends(clerk_guard),
):
```

- FastAPI parses the JSON body into the `Visit` Pydantic model:

```python
class Visit(BaseModel):
    patient_name: str
    date_of_visit: str
    notes: str
```

- `fastapi-clerk-auth` extracts the Bearer token from `Authorization`
- Fetches Clerk's public JWKS from `CLERK_JWKS_URL` and verifies the JWT signature
- Rejects the request with **403 Forbidden** if the token is invalid or expired
- On success, `creds.decoded["sub"]` contains the user's ID (available for auditing)

---

### 8. Backend Builds the Prompt

**File**: `api/index.py`

```python
system_prompt = """
You are provided with notes written by a doctor from a patient's visit.
Your job is to summarize the visit for the doctor and provide an email.
Reply with exactly three sections with the headings:
### Summary of visit for the doctor's records
### Next steps for the doctor
### Draft of email to patient in patient-friendly language
"""

def user_prompt_for(visit: Visit) -> str:
    return f"""Create the summary, next steps and draft email for:
Patient Name: {visit.patient_name}
Date of Visit: {visit.date_of_visit}
Notes:
{visit.notes}"""
```

- The system prompt instructs the model to produce exactly three structured sections
- The user prompt injects the form data from the POST body

---

### 9. Backend Calls OpenAI with Streaming

**File**: `api/index.py`

```python
client = OpenAI(api_key=openai_key, max_retries=0, timeout=8.0)
stream = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=prompt,
    stream=True,
)
```

- Uses `gpt-4o-mini` — fast, cost-effective, widely available
- `max_retries=0` and `timeout=8.0` prevent hanging within Vercel's 10s function limit
- OpenAI streams the response as a generator of delta chunks

---

### 10. Backend Streams SSE Response (with Error Handling)

**File**: `api/index.py`

```python
def event_stream():
    openai_key = os.getenv("OPENAI_API_KEY")
    if not openai_key:
        yield "data: **ERROR: OPENAI_API_KEY is not set on this deployment.**\n\n"
        return

    try:
        # ... OpenAI call ...
        for chunk in stream:
            text = chunk.choices[0].delta.content
            if text:
                lines = text.split("\n")
                for line in lines[:-1]:
                    yield f"data: {line}\n\n"
                    yield "data:  \n"
                yield f"data: {lines[-1]}\n\n"
    except Exception as e:
        yield f"data: **ERROR: {type(e).__name__}: {str(e)}**\n\n"

return StreamingResponse(event_stream(), media_type="text/event-stream")
```

- The response is **always** `text/event-stream` — errors are streamed as SSE messages, not HTTP exceptions
- This prevents the `@microsoft/fetch-event-source` library from throwing a content-type mismatch error
- Any OpenAI error (auth, timeout, model error) appears directly in the app's output area
- Multi-line chunks are split so each line is its own SSE event

---

### 11. Frontend Accumulates Chunks and Re-renders

**File**: `pages/product.tsx`

```tsx
onmessage(ev) {
    buffer += ev.data;
    setOutput(buffer);
}
```

- Each SSE message appends to `buffer`
- `setOutput(buffer)` triggers a React re-render on every chunk — streaming effect

---

### 12. ReactMarkdown Renders the Three Sections

**File**: `pages/product.tsx`

```tsx
<ReactMarkdown remarkPlugins={[remarkGfm, remarkBreaks]}>
    {output}
</ReactMarkdown>
```

- `ReactMarkdown` converts accumulated markdown to HTML on every re-render
- `remarkGfm` — GitHub Flavored Markdown (tables, strikethrough, etc.)
- `remarkBreaks` — converts single newlines to `<br>` tags
- The three sections (`### Summary`, `### Next steps`, `### Draft email`) appear incrementally as chunks arrive

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
<Protect plan="premium_subscription"> → shows <PricingTable> or <ConsultationForm>
    ↓
User fills in patient name, date, consultation notes → clicks Generate Summary
    ↓
handleSubmit → getToken() fetches JWT
    ↓
fetchEventSource POST /api  { Authorization: Bearer <jwt>, body: { patient_name, date_of_visit, notes } }
    ↓
FastAPI parses Visit model + verifies JWT via Clerk JWKS
    ↓
Builds system + user prompt with consultation notes
    ↓
OpenAI gpt-4o-mini streams delta chunks → FastAPI formats as SSE
    ↓
Frontend buffer accumulates chunks → setOutput() → re-render
    ↓
ReactMarkdown renders three sections: Summary / Next Steps / Patient Email
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
| `@microsoft/fetch-event-source` | ^2.0.1 | SSE client supporting POST + custom headers |
| `react-datepicker` | ^9.1.0 | Date picker for the visit date field |
| `react-markdown` | ^10.1.0 | Renders markdown string as HTML |
| `remark-gfm` | ^4.0.1 | GitHub Flavored Markdown plugin |
| `remark-breaks` | ^4.0.0 | Converts `\n` to `<br>` |
| `@tailwindcss/typography` | ^0.5.19 | Tailwind prose styling for markdown output |
| `tailwindcss` | ^4 | Utility-first CSS framework |
| `typescript` | ^5 | TypeScript compiler |

### Backend (Python) — `requirements.txt`

```bash
pip install -r requirements.txt
```

| Package | Purpose |
|---------|---------|
| `fastapi` | Web framework — defines the `POST /api` route |
| `uvicorn` | ASGI server to run FastAPI locally |
| `openai` | Official OpenAI Python SDK (streaming support) |
| `fastapi-clerk-auth` | FastAPI dependency that verifies Clerk JWTs via JWKS |
| `pydantic` | Data validation — `Visit` model for the POST body |

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
| `pages/index.tsx` | Landing page — MediNotes Pro hero + sign-in |
| `pages/product.tsx` | Protected page — plan gate + consultation form + SSE client |
| `api/index.py` | API endpoint — JWT verification, prompt building, SSE streaming |
| `requirements.txt` | Python dependencies |
| `.env.local` | Environment variables (not committed to git) |
| `next.config.ts` | Next.js configuration |
