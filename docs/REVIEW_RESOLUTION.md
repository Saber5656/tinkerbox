# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the three existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTN39pc6OZCME — Vite/PWA base path

Problem: deployment under `/tinkerbox/` can break assets, manifest, and service-worker scope when paths assume root.

Resolution: production configuration MUST set Vite `base: '/tinkerbox/'`; manifest `scope` and `start_url`, service-worker registration, and asset URLs MUST remain under that prefix.

Focused verification before resolving this thread: build and inspect emitted assets, manifest, and service-worker scope for the exact prefix.

## PRRT_kwDOTN39pc6OZCMH — production versus development CSP

Problem: a strict `default-src 'self'` policy can break Vite development inline styles and HMR if applied indiscriminately.

Resolution: production MUST use the strict documented CSP, while development MUST use a separately documented HMR-compatible policy. The development exception MUST NOT ship in production, and the production build/deployment gate MUST verify the strict policy is the one delivered.

Focused verification before resolving this thread: run the dev server/HMR path and inspect the production artifact/response separately; assert each environment receives its intended policy.

## PRRT_kwDOTN39pc6OZCMJ — DOM-free core boundary

Problem: a renderer that uses `ImageData`/`putImageData` cannot remain in a DOM-free core.

Resolution: the core MUST remain a pure grid/engine layer without DOM, Canvas, `ImageData`, or `putImageData` dependencies. The Canvas renderer MUST live under an app/render boundary (or equivalent adapter), with an explicit data contract to the core. Core lint/compile checks MUST reject DOM APIs.

Focused verification before resolving this thread: compile/lint the core in a non-DOM environment, inspect dependency boundaries, and run renderer tests only at the app boundary.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.