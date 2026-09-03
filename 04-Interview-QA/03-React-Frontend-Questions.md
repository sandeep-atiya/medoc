# React / Frontend Interview Questions — answered with your projects

Rule: give the concept in one or two sentences, then say where you used it.

---

## Core React

**Q1. What is the virtual DOM and reconciliation?**
React keeps a lightweight tree in memory, diffs it against the previous tree when state changes, and applies only the minimal DOM updates. *Where:* every SPA; matters in the ViewReports tables where thousands of rows change with filters — I combined it with virtualisation so only visible rows render.

**Q2. Function components and hooks you use most?**
`useState`, `useEffect`, `useRef`, `useMemo`, `useCallback`, `useReducer`, `useContext`, plus custom hooks. *Where:* `useCustomerSearch` (Patient-CRM) uses `useRef` for a monotonically increasing request id to drop stale responses; `useDraftAutosave` (Appointment-CRM) debounces form drafts; `useCohortData` (Cohorts) wraps fetch + polling.

**Q3. Explain `useEffect` cleanup.**
Return a function to cancel timers/subscriptions. *Where:* polling `setInterval` for sync status (Cohorts/ViewReports) is cleared on unmount; Socket.IO listeners are removed in `SocketContext` cleanup.

**Q4. How do you avoid unnecessary re-renders?**
Memoise expensive derived data (`useMemo`), stable callbacks (`useCallback`), split state, virtualise long lists, lazy-load routes. *Where:* ViewReports uses `React.lazy` + `Suspense` for every page and TanStack Virtual for tables; React Compiler (babel plugin) is enabled in Vite 8 projects to auto-memoise.

**Q5. What is React 19 new for you?**
Better form actions and the React Compiler support; I used the compiler plugin in Vite 8 projects (ViewReports, QMS, Cohorts, Email-Extract, Patient-CRM).

**Q6. Controlled vs uncontrolled inputs?**
Controlled: value from state. Uncontrolled: ref reads DOM. *Where:* react-hook-form (uncontrolled under the hood, performant) in ViewReports and Email-Extract; simple controlled inputs in chat input and search bars.

**Q7. How do you handle forms and validation?**
react-hook-form + zod schemas (ViewReports, Email-Extract); JSON-schema-driven dynamic forms with a generic renderer for 16 disease consultation forms (Appointment-CRM).

**Q8. Error boundaries?**
Class component catching render errors to show a fallback. *Where:* recommended addition; today I rely on route-level fallbacks and API error toasts (sonner / react-toastify).

**Q9. Keys in lists — why?**
Stable identity for reconciliation. *Where:* message SIDs in chat lists, call IDs in call tables.

---

## State management

**Q10. When Redux Toolkit vs Context vs Zustand vs React Query?**
- Redux Toolkit when many slices and cross-page state with persistence: Appointment-CRM (7 slices + redux-persist for auth/ui), Email-Extract (auth/ui).
- Zustand for small global stores with less boilerplate: ViewReports auth store.
- Context + useReducer for a single concern: Cohorts auth, Appointment-CRM FlowContext.
- TanStack React Query for **server state** (caching, refetch, stale time): ViewReports, QMS, Email-Extract.
Rule I follow: server data → React Query; client/UI state → Redux/Zustand/Context.

**Q11. What does redux-persist do and what did you persist?**
Saves selected slices to localStorage and rehydrates on load. I persisted only `auth` and `ui`, never data slices, so a refresh keeps the login but data stays fresh.

**Q12. React Query: staleTime vs cacheTime (gcTime)?**
staleTime = how long data is fresh (no refetch); gcTime = how long unused data stays in memory. I used staleTime 30 s with `retry: 1` and no refetch on window focus for dashboards.

**Q13. How did you sync filters with the URL?**
`nuqs` in ViewReports so report filters are shareable links; `replaceState` with `?q=` in Patient-CRM for deep links inside the dialer iframe.

---

## Routing

**Q14. React Router: v6 vs v7 — how did you structure routes?**
`createBrowserRouter` with lazy routes in ViewReports (v7); `<Routes>` style in WhatsApp and CRM (v6/v7). Protected routes via a `ProtectedRoute` wrapper that checks token/role; admin-only report pages check roles.

**Q15. How does SPA routing work on IIS/Apache?**
The server must return `index.html` for unknown paths: IIS URL Rewrite rule (`web.config`) or Apache `.htaccess` rewrite, excluding `/api` and real files.

---

## Data fetching & API layer

**Q16. How do you structure API calls?**
One axios instance per app (`apiClient.js`) with interceptors: attach JWT or `x-api-key`, unwrap `res.data`, on 401 clear token and redirect, on 403 emit a window event. Feature-level `*.api.js` modules call it.

**Q17. How do you handle race conditions in search?**
Request id ref; ignore responses that are not the latest (Patient-CRM). Also `AbortController` is a good alternative.

**Q18. How do you handle real-time updates?**
Socket.IO client in a Context provider with auto-reconnect; events update local state and unread badges; a status indicator shows connection state (WhatsApp Dashboard).

**Q19. How do you show long-running job progress?**
Poll a status endpoint every 3–5 s (sync status, scrape jobs) and stop polling when finished; show progress per source (ViewReports sync UI).

---

## Performance & build

**Q20. Vite — why, and what did you configure?**
Fast dev server and Rollup/Rolldown builds. Configured build modes (`--mode staging/production`), env files per mode (`VITE_API_URL`), manual chunks (vendor/charts/icons), drop console in production, dev proxy for `/api`.

**Q21. Code splitting?**
`React.lazy` per route + Suspense fallback; manual vendor chunks.

**Q22. Large tables?**
TanStack Table for column logic + TanStack Virtual to render only visible rows; server-side pagination for call history.

**Q23. Caching static assets?**
Hashed filenames from Vite; IIS `clientCache` 365 days for `/assets`, `no-cache` for `index.html`; Apache mod_expires on Cloudways.

---

## Styling & UI

**Q24. Tailwind v3 vs v4 differences you met?**
v4 uses the Vite plugin and CSS-first config (`@import "tailwindcss"`, no `tailwind.config.js`); v3 needs PostCSS + config. I used v3 in the CRM/WhatsApp and v4 in ViewReports/QMS/Email-Extract.

**Q25. What is the shadcn-style approach?**
Copy small primitives (button, card, badge, input) into the repo built with `class-variance-authority` + `clsx` + `tailwind-merge`, so you own the components and can theme them.

**Q26. Accessibility basics you follow?**
Semantic elements, labels for inputs, focus states, keyboard-friendly OTP boxes with auto-advance, sufficient contrast in heatmap colour bands.

---

## Iframe / embedding (unique to you)

**Q27. What problems did you face embedding React apps in another app?**
- Headers: remove `X-Frame-Options`, set `Content-Security-Policy: frame-ancestors *` (or a specific host) in `web.config`.
- Auth: values arrive via URL or `postMessage` after first render → ProtectedRoute polls storage for up to 2 s.
- Cookies are unreliable in third-party iframes → header-based session instead.
- Deep-link mode when `?q=` or `?callId=` is present to hide the shell UI.

---

## Testing & quality (frontend)

**Q28. How do you test the UI?**
Manual + UAT with real users inside the real host (dialer, portal); ESLint with react-hooks and react-refresh plugins; I would add Vitest + React Testing Library for hooks and components next.

---

## Rapid-fire definitions

- **Props vs state:** props come from parent (read-only); state is local and changeable.
- **Lifting state up:** move shared state to the nearest common parent.
- **Prop drilling:** passing props through many levels → solve with Context/store.
- **Memoisation:** cache a computed value/component until inputs change.
- **Hydration:** attaching React to server-rendered HTML (not used — all my apps are client-rendered SPAs).
- **CSR vs SSR:** my apps are CSR behind IIS/Apache; SSR (Next.js) would help SEO — not needed for internal tools.
- **Debounce vs throttle:** debounce waits for a pause (search box, autosave); throttle limits rate (scroll).
- **CORS from the browser side:** the browser blocks cross-origin responses unless the server sends `Access-Control-Allow-Origin`; I solved it with an allowlist in Express or by reverse-proxying `/api` on the same origin (IIS ARR, Apache).
