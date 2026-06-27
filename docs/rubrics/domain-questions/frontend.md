# Frontend Domain Questions

## React & Component Architecture

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 1 | Explain the React component lifecycle in functional components (hooks). | SDE 2 | `useState` (state), `useEffect` (mount/update/unmount via cleanup), `useRef` (mutable ref without re-render), `useMemo`/`useCallback` (memoization). Effect deps array controls when effects fire. | What causes infinite re-render loops? How do you debug stale closures? |
| 2 | When would you use `useReducer` over `useState`? Design a complex form state with it. | SDE 2 | `useReducer` for: complex state logic, multiple sub-values, next state depends on previous. Dispatch actions like `SET_FIELD`, `VALIDATE`, `RESET`. Centralizes state transitions. | How do you handle async actions with useReducer? Compare to Redux Toolkit slice. |
| 3 | Explain React's reconciliation algorithm. Why are keys important in lists? | SDE 2-3 | Virtual DOM diffing: same type → update props; different type → unmount + remount. Keys identify elements across renders; without keys, React re-renders entire list on reorder. Index-as-key breaks animations and input state. | What is React Fiber? How does concurrent mode change reconciliation? |

## State Management

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 4 | Compare Redux, Zustand, Jotai, and React Query. When would you pick each? | SDE 2-3 | Redux: complex app-wide state, time-travel debugging. Zustand: simpler global state, less boilerplate. Jotai: atomic state, fine-grained reactivity. React Query: server state (cache, sync, optimistic updates). Key insight: separate server state from client state. | How do you handle optimistic updates with rollback? When does Context API suffice? |
| 5 | How would you manage authentication state across micro-frontends? | SDE 3 | Shared auth module (singleton); JWT in httpOnly cookie (not localStorage for XSS); token refresh via silent iframe or interceptor. Cross-origin: shared cookie domain or auth event bus. | How do you handle token expiry during active use? CSRF protection strategy? |

## Rendering & Performance

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 6 | How do you identify and fix unnecessary re-renders in React? | SDE 2 | React DevTools Profiler; `React.memo` for pure components; `useMemo`/`useCallback` for stable references; lift state down; split contexts. Avoid: inline objects/functions in JSX props. | What's the cost of over-memoizing? When does React.memo hurt? |
| 7 | Explain SSR vs CSR vs SSG vs ISR. When would you use each? | SDE 2-3 | CSR: SPA, fast interactions, poor initial SEO. SSR: dynamic content, good SEO, higher TTFB. SSG: static content, fastest load, build-time only. ISR: static + periodic revalidation (Next.js). | How do you hydrate SSR content? What's the hydration mismatch problem? Streaming SSR benefits? |
| 8 | How do you optimize a React app that takes 8 seconds to load? | SDE 2-3 | Code splitting (React.lazy + Suspense); tree shaking; bundle analysis (webpack-bundle-analyzer); lazy-load below-fold; compress assets (Brotli); prefetch critical resources; reduce third-party scripts. | How do you measure Core Web Vitals? What's the impact of third-party scripts? |

## Accessibility

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 9 | How do you make a custom dropdown component accessible? | SDE 2 | ARIA roles (`listbox`, `option`); keyboard navigation (arrow keys, Enter, Escape); focus management; `aria-expanded`, `aria-activedescendant`; screen reader announcements via `aria-live`. | How do you test accessibility? What's the difference between aria-label and aria-labelledby? |

## Build Tools & Bundling

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 10 | Explain how tree shaking works. Why might it fail? | SDE 2-3 | Static analysis of ES module imports/exports; removes unused exports. Fails with: CommonJS (dynamic), side effects in modules (mark with `sideEffects: false`), barrel files re-exporting everything, dynamic imports. | How does Vite's approach differ from Webpack? What are the benefits of ESM-first bundling? |
