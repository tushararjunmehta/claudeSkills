---
name: Write Tests
description: This skill should be used when the user asks to "write tests", "add tests", "create unit tests", "test this component", or "add test coverage".
version: 1.0.0
---

## Testing Stack

- **Test runner:** Vitest (globals enabled — no need to import `describe`, `it`, `expect`, `vi`)
- **Component rendering:** `@testing-library/react`
- **User interactions:** `@testing-library/user-event` v14
- **API mocking:** MSW v2 (`msw/node`)
- **DOM matchers:** `@testing-library/jest-dom` (loaded via `src/test-setup.ts`)

Run tests with: `npm run test`

---

## Conventions

### 1. Always wrap renders in `<UserProvider>`

Every component that uses `useUser()` will throw without it.

```tsx
import { UserProvider } from `@/lib/UserContext`

render(<UserProvider><MyComponent /></UserProvider>)
```

### 2. Use `userEvent` for interactions

```tsx
import { userEvent } from `@testing-library/user-event`

await userEvent.click(screen.getByRole(`button`, { name: `Submit` }))
await userEvent.type(screen.getByPlaceholderText(`Mobile Number`), `9876543210`)
```

### 3. Use `waitFor` for anything async

After clicking a button that triggers an API call, the DOM update is async. Use `waitFor` to retry the assertion until it passes.

```tsx
import { waitFor } from `@testing-library/react`

await waitFor(() => expect(screen.getByText(`Success`)).toBeInTheDocument())
```

### 4. Use `vi.fn()` to spy on prop callbacks

```tsx
const onClose = vi.fn()
render(<UserProvider><LoginModal isOpen={true} onClose={onClose} /></UserProvider>)
await userEvent.click(screen.getByRole(`button`, { name: `Close` }))
expect(onClose).toHaveBeenCalledTimes(1)
```

---

## API Mocking with MSW

### MSW server setup (already exists — do not recreate)

- Server: `src/mockServices/server.ts`
- Default handlers: `src/mockServices/handlers.js`
- Setup file: `src/test-setup.ts` — starts/stops server and resets handlers after each test

### Adding runtime mocks inside a test

```tsx
import { server } from `@/mockServices/server`
import { http, HttpResponse } from `msw`

server.use(
    http.post(`https://uat.lockthedeal.com/api/mobile/otp.json`, () =>
        HttpResponse.json({ success: true }, { status: 200 })
    )
)
```

**Critical:** Mock URLs must use `https://uat.lockthedeal.com` (from `NEXT_PUBLIC_API_URL` in `.env.local`). Using `http://localhost/...` will not intercept — the real backend will be called instead.

### Handlers reset automatically

`afterEach(() => server.resetHandlers())` is in `test-setup.ts`. Runtime handlers added via `server.use()` are cleared after each test — no manual cleanup needed.

---

## Pattern to reach OTP step in tests

```tsx
server.use(
    http.post(`https://uat.lockthedeal.com/api/mobile/validateMobile.json`, () =>
        HttpResponse.json({ data: { name: `Tushar` } }, { status: 200 })
    ),
    http.post(`https://uat.lockthedeal.com/api/mobile/otp.json`, () =>
        HttpResponse.json({ success: true }, { status: 200 })
    )
)

render(<UserProvider><LoginModal isOpen={true} onClose={() => {}} /></UserProvider>)
await userEvent.type(screen.getByPlaceholderText(`Mobile Number`), `9876543210`)
await userEvent.click(screen.getByRole(`button`, { name: `LOGIN via OTP` }))
await waitFor(() => expect(screen.getByPlaceholderText(`Enter Your OTP`)).toBeInTheDocument())
```