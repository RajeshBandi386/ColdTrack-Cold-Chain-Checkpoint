# ColdTrack: Cold-Chain Checkpoint App

A driver app for temperature-sensitive medicine deliveries (vaccines & insulin) on routes with unreliable connectivity.

Built following **Feature-based Clean Architecture**, **Strict TypeScript**, **TanStack Query**, **Zustand (persisted)**, **React Hook Form + Zod**, and an **in-memory Fake API** with realistic misbehaviour (latency, 500 errors, client timeouts, idempotency, version conflicts).

---

## 1. Quick Start

```bash
# Install dependencies
npm install

# Start development server on port 3000
npm run dev

# Run TypeScript typechecks
npm run lint

# Build for production
npm run build
```

---

## 2. Architecture & Layer Separation

The codebase follows the strict Clean Architecture guidelines laid out in Section 4:

```
src/
├── app/                  # Application Shell, Providers (QueryClient), Navigation
├── core/
│   ├── network/          # fakeApi.ts, httpClient.ts, errors.ts (typed error hierarchy)
│   ├── storage/          # Typed storage wrapper (persists across app restarts)
│   ├── sync/             # syncQueue.store.ts (Zustand + persist), syncEngine.ts
│   └── utils/            # clock.ts (injectable IClock with FakeClock), id.ts
├── features/
│   ├── shipments/
│   │   ├── domain/       # Pure TypeScript: entities, rules, schemas (Zod), use cases
│   │   ├── data/         # shipment.repository.ts, dto.ts, mappers.ts
│   │   └── presentation/ # screens/, components/, hooks/ (useShipments, useShipmentDetail, useAddReading)
│   ├── handover/
│   │   ├── domain/       # entities, rules, schemas (Zod), submitHandover.usecase.ts
│   │   ├── data/         # handover.repository.ts, dto.ts, mappers.ts
│   │   └── presentation/ # handoverDraft.store.ts, useHandover, HandoverWizardScreen
│   └── sync/
│       └── presentation/ # SyncCenterScreen, GlobalSyncBanner, TestRunnerModal
└── tests/
    └── testSuite.ts      # Automated unit tests covering 6 domain tests + 1 sync engine test
```

### Dependency Rules Adhered To
1. **Pure Domain Layer**: `domain/` contains zero React, React Native, TanStack, Zustand, or storage imports. Completely unit-testable.
2. **Injectable Clock**: All domain business rules use `clock.now()` / `clock.nowSeconds()`, never hardcoded `Date.now()`, enabling deterministic temporal testing (e.g. "until now" open-ended excursions).
3. **DTO Separation**: `data/` owns DTOs and mappers. Domain entities never contain raw API field names (`shipment_id`, `min_temp_deci_c`, `temp_deci_c`, `status_code` vs `id`, `minTemperatureC`, `status`).
4. **Presentation Boundaries**: Presentation screens never call the API directly. Flow: Screen -> Hook -> Use Case -> Repository.

---

## 3. Real-Life Scenarios Handled (Section 7)

All 9 scenarios from Section 7 have been handled and can be verified by hand in the app:

| Scenario | Requirement | How It Is Implemented |
|---|---|---|
| **A** | Go offline, add 3 readings, kill app, reopen, go online | Toggle "Offline" in top banner. Add 3 readings; they show as pending in timeline. Reload/restart page: readings survive via persisted Zustand store. Toggle "Online": sync engine processes FIFO in exact creation order. |
| **B** | Double-tap "Save reading" quickly | `useAddReading` implements an atomic ref lock and disables submission while executing to guarantee exactly one reading is created. |
| **C** | Write times out on client but succeeded on server; app retries | The fake API randomly simulates client timeouts after saving. On retry, the sync engine re-uses the exact same `idempotencyKey`; server detects the existing key and returns the cached 200 without creating a duplicate. |
| **D** | Server returns 500 repeatedly | Exponential backoff (`[1s, 2s, 4s, 8s, 16s]`) across 5 attempts, transitioning to `failed` with manual "Retry". Independent per-shipment processing ensures other shipments keep moving. |
| **E** | Dispatcher cancels shipment while driver offline | Reviewer test button bumps server version or marks status `CX`. On sync, `ConflictError (409)` is caught and surfaced in a "Conflict / Needs attention" state without infinite retrying. |
| **F** | Pull-to-refresh while readings pending | Repository and mappers merge pending queued readings with fetched server data before passing to the domain layer. Pull-to-refresh never hides unsynced readings. |
| **G** | Driver fills handover step 2, app is killed | `useHandoverDraftStore` persists draft state keyed by `shipmentId`. Reopening resumes at the exact step with all fields intact. Cleared upon final confirmation. |
| **H** | Out-of-range pushes excursion > 30 min | `isShipmentCompromised` dynamically computes cumulative excursion time. When it crosses 30m, status flips to `COMPROMISED` across List, Detail, and Handover immediately. |
| **I** | Device timezone differs from UTC | All timestamps in DTOs and storage are UTC epoch seconds; all presentation dates are formatted in local device timezone. |

---

## 4. Tests (Section 8)

The app includes an automated test runner (accessible via the **"Verify Tests"** or **"Run Tests (6+1)"** button in the header) executing all 7 required tests:

1. **Excursion Minutes Calculation**: Multi-segment calculation including the open-ended "until now" case with `FakeClock`.
2. **Compromised Flag Derivation**: Correct boundary check at 30 minutes (25 min safe, 31 min compromised).
3. **Chronology Rule Validation**: Enforces no reading before dispatch, before previous reading, or >2 min in the future.
4. **Phone Normalization Schema**: Strips `+91`, spaces, dashes and verifies 10 digits starting with 6–9.
5. **Licence Format Schema**: Enforces `AA-PH-12345` with auto-uppercase.
6. **Conditional Note Schema**: Optional in-range, but strictly requires minimum 10 characters during temperature excursion.
7. **Bonus: Sync Engine Idempotency**: Simulates repeated writes with the same idempotency key and proves zero duplicate creation on the server.

---

## 5. Decisions & Trade-offs

- **Virtualized windowing for 500 shipments**: The fake API generates 500 cargo records. To prevent DOM bloat while keeping scrolling smooth, a lightweight windowed feed with search and status filtering is implemented.
- **Offline FIFO per shipment**: Instead of a global lock that stops all syncing when one item fails, the sync engine groups queued operations by `shipmentId`. If shipment A encounters a 500 error or conflict, shipment B continues syncing uninterrupted.
- **Human-centric Cold-Chain UX**: High-contrast gauges, clear temperature bands, unmistakable compromised alerts, and adequate touch targets ensure the driver can read and tap quickly under bright sunlight and bumpy road conditions.
