---
name: Project Conventions
description: This skill should be used when the user asks about "how does the login flow work", "how does OTP work", "how to use apiClient", "explain UserContext", "how is auth handled", "where is the token stored", "how does the API proxy work", or works on any new component or feature in this project.
version: 1.0.0
---

## Auth Flow (Mobile OTP Login)

The login flow is in `src/components/LoginModal.tsx` and has two steps:

**Step 1 — Phone input:**
1. Validates phone is exactly 10 digits
2. Calls `POST /api/mobile/validateMobile.json` — extracts user name from response (success or 401 catch)
3. Calls `POST /api/mobile/otp.json` — triggers OTP send to phone
4. On success: transitions to OTP step (`setIsOtpStep(true)`)

**Step 2 — OTP input:**
1. Validates OTP is at least 4 digits
2. Calls `PUT /api/mobile/verifyotp.json`
3. On success: calls `login(userData, authToken)` from `UserContext` and `onClose()`

### Why `validateMobile.json` returns 401 for existing users

A 401 from `validateMobile.json` is NOT an error — it means the number is already registered. The user's name comes from `e?.response?.data?.data?.name` in the catch block. The code then proceeds to send the OTP normally.

---

## apiClient (`src/lib/apiClient.ts`)

- Axios instance with `baseURL = process.env.NEXT_PUBLIC_API_URL` (`https://uat.lockthedeal.com`)
- Auth token is attached automatically from `localStorage.getItem('token')` as `Authorization: Bearer <token>` on every request
- All `apiService` methods (`post`, `get`, `put`, `delete`) return `res.data` — the response body directly, NOT the full axios response object

```ts
// res.data is the body, not the full response
const result = await apiService.post("/some/endpoint", payload)
// result === response body directly
```

- Common headers include `version: 3500000` — some endpoints delete this via `transformRequest`

---

## UserContext (`src/lib/UserContext.tsx`)

Provides auth state globally via React Context.

```ts
const { user, token, login, logout, isAuthenticated } = useUser()
```

| Property | Type | Description |
|---|---|---|
| `user` | `User \| null` | `{ id, firstName, fullName, username, mobile }` |
| `token` | `string \| null` | JWT access token |
| `login(userData, authToken)` | function | Sets state + saves to localStorage |
| `logout()` | function | Clears state + removes from localStorage |
| `isAuthenticated` | boolean | `true` if token exists |

**localStorage keys:** `user` (JSON stringified), `token` (raw string)

**Important:** `useUser()` must be called inside `<UserProvider>`. Calling it outside throws: `"useUser must be used within a UserProvider"`.

---

## API Proxy (Route Handler)

The backend (`uat.lockthedeal.com`) requires a system-level `AUTH_TOKEN` on all `/api/mobile/` requests. This token must never reach the browser.

A Next.js Route Handler at `src/app/api/mobile/[...path]/route.ts` proxies these requests server-side, injecting `AUTH_TOKEN` from `.env.local` before forwarding to the real backend.

**Why this matters:** `NEXT_PUBLIC_API_URL` points to the real backend, but `/api/mobile/` calls go through the local Next.js server first (same-origin) to keep `AUTH_TOKEN` server-side only.

---

## Environment Variables (`.env.local`)

| Variable | Value | Usage |
|---|---|---|
| `NEXT_PUBLIC_API_URL` | `https://uat.lockthedeal.com` | axios baseURL |
| `NEXT_PUBLIC_API_TOKEN` | `Bearer eyJ...` | Fallback token for non-browser contexts |

---

## File Locations

| File | Purpose |
|---|---|
| `src/components/LoginModal.tsx` | Mobile OTP login modal |
| `src/components/LoginModal.test.tsx` | Tests for LoginModal |
| `src/lib/UserContext.tsx` | Auth context — user, token, login, logout |
| `src/lib/apiClient.ts` | Axios instance + apiService helpers |
| `src/lib/types.ts` | Shared TypeScript types |
| `src/lib/api/product.ts` | Product API calls |
| `src/mockServices/server.ts` | MSW server instance (tests only) |
| `src/mockServices/handlers.js` | Default MSW handlers (tests only) |
| `src/test-setup.ts` | Global test setup |
| `src/app/api/mobile/[...path]/route.ts` | Next.js Route Handler — proxies mobile API calls |
