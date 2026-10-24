# Implementation Plan: AI Furniture Finder & Room Visualizer

## Overview

This plan builds the application in four independently testable MVP phases, matching the design's phased rollout. Phase 1 stands up the Next.js foundation, the provider abstraction with DEMO fallbacks, and the full input wizard. Phase 2 adds real product search with the normalize/filter/rank pipeline and honest fit assessment. Phase 3 adds the asynchronous AI visualization pipeline with per-product failure isolation. Phase 4 adds comparison, favorites/accounts, performance, and end-to-end DEMO integration tests.

Implementation language is **TypeScript** (Next.js App Router + React + Tailwind on the client; Next.js server routes + a DB-backed job worker on the server). Testing uses **Jest + fast-check**. Property tests encode correctness properties P1–P8 from the design and are placed next to the code they validate so regressions surface early. Each phase boots and runs end-to-end in DEMO mode without any external credentials.

## Tasks

### Phase 1 — Website Foundation

- [x] 1. Scaffold the Next.js project and shared domain types
  - Initialize a Next.js (App Router) + React + TypeScript + Tailwind CSS project with ESLint/Prettier and a `src/` layout
  - Configure Jest + ts-jest and fast-check; add `test` and `test:watch`-free single-run scripts
  - Create `src/domain/types.ts` with shared value types from the design: `FitStatus`, `ProviderMode`, `SpecCompleteness`, `MeasurementConfidence`, `Dimensions`, `Money`, `RankedResult`, `FurnitureCategory`, `PlacementArea`, `LengthUnit`, `Material`, `InteriorStyle`, `ProductCondition`
  - Create `src/domain/models.ts` with `RoomProject`, `SearchCriteria`, `Product`, `Visualization`, `StrictFieldFlags`, `ImageRef`
  - _Requirements: 13.7_

- [ ] 2. Define provider abstraction interfaces and the ProviderRegistry
  - [~] 2.1 Define provider interfaces
    - Create `src/providers/interfaces.ts` with `VisionProvider`, `CommerceProvider`, `ImageGenProvider`, `PreferenceInterpreter`, and `ProviderRegistry` (with `capabilities(): SystemCapabilities`), matching the design's Components and Interfaces section
    - Define `SystemCapabilities` as a per-subsystem map of `ProviderMode`
    - _Requirements: 6.1, 6.2, 13.3_

  - [~] 2.2 Implement DEMO provider fallbacks
    - Implement `DemoCommerceProvider` (mode `DEMO`, returns clearly labeled `source: "demo"` sample products), `DemoVisionProvider`, and `DemoImageGenProvider` under `src/providers/demo/`
    - Every demo product field is sample data; none is presented as a live result
    - _Requirements: 6.2, 6.3, 6.4_

  - [~] 2.3 Implement ProviderRegistry credential resolution
    - Implement `src/providers/registry.ts` that reads `COMMERCE_*`, `VISION_*`, `IMAGEGEN_*` env vars at init, binds a LIVE provider when credentials are present and validate, otherwise binds the matching `Demo*` provider, and exposes `capabilities()`
    - Ensure no credential value is ever placed on the returned capabilities payload
    - _Requirements: 6.1, 6.2, 13.1, 13.3, 13.4_

  - [ ]* 2.4 Write property test for mode honesty (P8)
    - **Property 8: Mode honesty** — for random presence/absence of each credential set, assert `capabilities().<subsystem>` is `DEMO` exactly when that subsystem's credentials are absent/invalid, and `LIVE` otherwise
    - **Validates: Requirements 6.1, 6.2, 6.3, 13.3, 13.4**

  - [ ]* 2.5 Write property test for credential custody (P7)
    - **Property 7: Credential custody** — serialize the `capabilities()` payload and any client-bound registry output; assert no configured credential string ever appears
    - **Validates: Requirements 13.1, 13.2**

- [~] 3. Build the capabilities endpoint and demonstration-mode banner
  - Add `GET /api/capabilities` server route returning `registry.capabilities()` (server-side only)
  - Add a client `ModeBanner` component that renders an honest banner for any subsystem reporting `DEMO` ("Demonstration Mode — sample products shown")
  - _Requirements: 6.3, 13.3, 13.4_

- [~] 4. Build homepage and responsive app layout
  - Create the App Router root layout, global Tailwind styles, and a responsive homepage that introduces the workflow and links into the wizard
  - Mount the `ModeBanner` in the shared layout
  - _Requirements: 13.6, 13.7_

- [ ] 5. Implement room-image upload service
  - [~] 5.1 Implement upload validation and metadata stripping
    - Implement `src/services/upload/UploadService.ts` `validateUpload()` enforcing MIME (JPEG/PNG/WebP), size (>0 and ≤10 MB), and decodability; return specific type/size/corrupt messages
    - Implement EXIF/GPS stripping on accepted images before persistence
    - _Requirements: 1.1, 1.2, 1.3, 1.8, 1.11, 13.5_

  - [~] 5.2 Implement signed-URL storage and deletion
    - Implement `storeRoomImage()` returning a storage reference (never a raw public URL), `getSignedViewUrl()` with TTL clamped to ≤300 seconds, and `deleteImage()`
    - _Requirements: 1.9, 1.10_

  - [~] 5.3 Wire the upload API route and upload UI
    - Add `POST /api/upload` (multipart) that calls `UploadService`, persists only on success, and returns `{ roomImageRef }`
    - Build the client upload component supporting drag-and-drop and file picker, image preview, and crop/replace/remove controls
    - _Requirements: 1.4, 1.5, 1.6, 1.7, 13.6_

  - [ ]* 5.4 Write unit tests for upload validation and failure scenarios
    - Test accept path, unsupported type rejection, oversize rejection, corrupt-file rejection, and that no object persists on rejection; assert signed-URL TTL never exceeds 300s
    - _Requirements: 1.1, 1.2, 1.3, 1.10, 1.11_

- [~] 6. Implement furniture category and placement selection
  - Build the category selector (sofas, armchairs, coffee tables, dining tables, dining chairs, beds, desks, bookshelves, TV stands, wardrobes, nightstands, custom/other) and placement selector (`against_wall`, `beside_window`, `corner`, `center`)
  - Record selected category and `PlacementArea` on the room project; accept a user label when custom/other is chosen
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

- [ ] 7. Implement unit conversion and measurement validation core
  - [~] 7.1 Implement `toMillimeters` and measurement validation
    - Implement `src/domain/units.ts` `toMillimeters(value, unit)` with exact factors (mm, cm, m, in=25.4, ft=304.8) per the design
    - Implement `validateMeasurement(value, unit)`: positive, >0, ≤100000 in selected unit, ≤2 decimal places; return field-identifying messages
    - _Requirements: 4.2, 4.3, 4.4, 7.1, 13.5_

  - [ ]* 7.2 Write property test for unit round-trip (P5)
    - **Property 5: Unit round-trip** — for random non-negative values and supported units, assert `toMillimeters` equals the exact mathematical conversion and that filtering uses only canonical mm values
    - **Validates: Requirements 7.1, 7.2**

  - [ ]* 7.3 Write unit tests for measurement validation
    - Cover non-numeric, ≤0, >100000, and >2-decimal rejections; assert other entered values are retained
    - _Requirements: 4.3, 4.4_

- [ ] 8. Implement category-specific measurement form with reference anchoring
  - [~] 8.1 Implement `buildMeasurementForm` and confidence derivation
    - Implement `RoomAnalysisService.buildMeasurementForm(category)` producing category-appropriate fields
    - Implement `measurementConfidence` derivation: `confirmed` when a reference exists and all values are user-confirmed, `mixed` when a reference exists with at least one AI estimate, and never `confirmed` without a user-confirmed reference
    - _Requirements: 4.1, 4.9, 4.10, 4.11_

  - [~] 8.2 Build the measurement form UI with diagrams and navigation
    - Render dynamic fields with unit selector, per-field visual diagrams, value replacement on change, and back-navigation that retains entered values
    - Block search submission while no user-confirmed reference exists and show the required-reference message
    - _Requirements: 4.1, 4.2, 4.5, 4.6, 4.7, 4.8, 13.6_

  - [ ]* 8.3 Write property test for reference-anchored measurement (P6)
    - **Property 6: Reference-anchored measurement** — for random confirmed/estimated value sets, assert `measurementConfidence = confirmed` implies at least one user-confirmed reference exists, and no fit claim rests on estimation alone
    - **Validates: Requirements 4.8, 4.9, 4.11**

- [~] 9. Implement search filter UI and criteria validation
  - Build the filter form: required category + maximum dimensions; optional minimum dimensions, min/max price, colors, materials, styles, brands, min rating, min review count, condition, and free-text description
  - Implement per-field strict/preference toggles and validation: reject when `budgetMin > budgetMax`, and reject submission when any strict field is absent or malformed (identifying the field)
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 13.5_

- [ ] 10. Implement results page with DEMO-labeled products
  - [~] 10.1 Wire the DEMO search path end-to-end
    - Add `POST /api/search` that calls the (DEMO) `CommerceProvider` and returns results; add `GET /api/results` for revisiting
    - _Requirements: 6.3, 6.4, 13.6_

  - [~] 10.2 Build the responsive results gallery
    - Render a responsive grid (3 columns large, 2 medium, 1–2 small); each card shows product name, brand, price, key dimensions, color, material, rating, a FitStatus badge, a clear DEMO label when `source === "demo"`, and a View Product control; render cards without an in-room image when none is available
    - _Requirements: 9.1, 9.2, 9.6, 6.3, 13.7_

- [~] 11. Checkpoint — Phase 1 foundation
  - Ensure all tests pass, ask the user if questions arise.

### Phase 2 — Real Product Search

- [ ] 12. Implement the Amazon CommerceProvider
  - [~] 12.1 Implement `AmazonCommerceProvider`
    - Implement `src/providers/live/AmazonCommerceProvider.ts` (mode `LIVE`) calling Amazon PA-API; enforce rate limits, image-use policy, attribution, and affiliate disclosure on every request
    - Return only provider-sourced fields; never fabricate titles, specs, prices, ratings, or links; surface a provider-sourced error (no substitute products) on rate-limit/provider failure
    - _Requirements: 6.1, 6.5, 6.6, 6.10_

  - [ ]* 12.2 Write unit tests for commerce error and no-fabrication behavior
    - Test rate-limit/provider-error path returns a provider-sourced error and does not pad results; test that absent provider fields remain absent
    - _Requirements: 6.5, 6.10_

- [ ] 13. Implement product spec normalization
  - [~] 13.1 Implement `normalizeProduct`
    - Implement `src/services/pipeline/normalize.ts` converting every required dimension axis to canonical mm (exact factors, no rounding loss) and price to a single canonical currency; leave absent fields absent
    - _Requirements: 7.1, 7.2, 6.5_

  - [ ]* 13.2 Write property test for no fabrication (P4)
    - **Property 4: No fabrication** — fuzz raw provider payloads with missing fields; assert no normalized output field is non-null unless present in input
    - **Validates: Requirements 6.5, 7.1**

- [ ] 14. Implement the normalize/filter/rank pipeline with fit logic
  - [~] 14.1 Implement completeness, cap, and minimum checks
    - Implement `assessDimensionCompleteness`, `exceedsHardCap` (strict exceed on any present axis; absent axes not treated as fitting), and `belowMinimum` per the design's formal specs
    - _Requirements: 7.3, 7.4, 7.6_

  - [~] 14.2 Implement strict filters, FitStatus assignment, and ranking
    - Implement `FilterRankService.process(raw, criteria)`: apply mandatory budget + strict preference filters, assign `FITS` only when completeness is `complete` on all required axes and within caps (else `NEEDS_VERIFICATION`), rank by weighted score (dimensional fit, style, material, color, price, rating), and return distinct eligible products capped at the configured maximum
    - Ensure fit assessment uses verified specs and confirmed measurements only — no visualization input
    - _Requirements: 7.5, 7.6, 7.7, 7.8, 7.9, 12.1, 12.2_

  - [ ]* 14.3 Write property test for fit requires verification (P1)
    - **Property 1: Fit requires verification** — for random products and criteria, assert every result with `fitStatus === "FITS"` has `complete` dimensions on all required axes and is within `maximumDimensions` in canonical mm
    - **Validates: Requirements 7.3, 7.7, 12.2**

  - [ ]* 14.4 Write unit tests for dimension and budget filtering
    - Test `exceedsHardCap`/`belowMinimum` boundaries, mandatory budget filtering, strict-filter rejection, and that incomplete-dimension products become `NEEDS_VERIFICATION`; include the empty-result-set case
    - _Requirements: 7.3, 7.4, 7.5, 7.6, 12.2, 12.3_

- [ ] 15. Implement outbound link resolution
  - [~] 15.1 Implement `resolveOutboundUrl`
    - Implement resolution: Amazon + affiliate authorization → affiliate URL with disclosure metadata; Amazon without authorization → canonical listing URL; demo source → link flagged `demo:true`, never a live purchase path
    - _Requirements: 6.7, 6.8, 6.9_

  - [ ]* 15.2 Write unit tests for product-link handling
    - Cover affiliate, canonical-fallback, and demo-flagged branches and their disclosure/flag metadata
    - _Requirements: 6.7, 6.8, 6.9_

- [~] 16. Build product detail pages and empty-result handling
  - Add `GET /api/products/[id]` and a detail page showing the visualization slot, original room image, product specifications, fit assessment, and the resolved purchase link
  - When the pipeline returns no products, explain which constraints eliminated candidates and offer to relax a named constraint; re-run search on relaxation without restarting; present latest provider-sourced price/availability (no stale fabricated values)
  - _Requirements: 9.3, 9.5, 12.3, 12.4, 12.6, 13.6_

- [~] 17. Checkpoint — Phase 2 real search
  - Ensure all tests pass, ask the user if questions arise.

### Phase 3 — AI Visualization

- [ ] 18. Implement VisionProvider room analysis
  - [~] 18.1 Implement `VisionProvider` live binding and `RoomAnalysisService.analyze`
    - Implement a live `VisionProvider` and wire `analyze()` to detect walls/floor/windows/doors/corners/obstacles and return a candidate `PlacementArea` when no defect is reported
    - Handle defects (too dark, blurry, insufficient coverage): return the defect + photography guidance, withhold the candidate placement area, and expose a re-upload control; handle faces/sensitive content with a warning, re-upload control, and non-persistence after processing; handle provider failure with a retry/re-upload path
    - Label every AI-derived dimension value as an `estimate` contributing `estimated` confidence
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8_

  - [~] 18.2 Wire the analyze API route and room-analysis UI
    - Add `POST /api/rooms/analyze` returning features, measurement form, and `requiresReference: true`; render detected features, defect guidance, and re-upload/retry controls with loading states
    - _Requirements: 3.2, 3.3, 3.8, 13.6_

  - [ ]* 18.3 Write unit tests for vision failure scenarios
    - Cover each defect branch, the faces/sensitive-content branch (including non-persistence), and the provider-failure branch
    - _Requirements: 3.3, 3.4, 3.5, 3.6, 3.8_

- [~] 19. Implement the live ImageGenProvider
  - Implement a reference-conditioned live `ImageGenProvider` (mode `LIVE`) whose `generate()` throws on error/timeout/policy rejection so failures isolate per job
  - _Requirements: 8.3, 12.5_

- [ ] 20. Implement the async VisualizationOrchestrator with queue and worker
  - [~] 20.1 Implement enqueue and per-product job rows
    - Implement `VisualizationOrchestrator.enqueueForSearch` creating exactly one `pending` visualization row and one isolated job per eligible product
    - _Requirements: 8.1, 8.2_

  - [~] 20.2 Implement the worker with failure isolation and state machine
    - Implement `handleVizJob`: transition pending → generating, call `ImageGenProvider.generate`, store the image, set `completed` with the image ref (present only while `completed`); on error set only that job `failed` with a reason, leaving siblings unchanged; enforce monotonic transitions toward terminal states
    - _Requirements: 8.2, 8.3, 8.4, 12.5_

  - [~] 20.3 Implement cancel and retry
    - Implement `cancel` (pending/generating → canceled; already-terminal returns an "already terminal" indication) and `retry` (re-run a failed job independently)
    - _Requirements: 8.5, 8.6, 8.7_

  - [ ]* 20.4 Write property test for failure isolation (P3)
    - **Property 3: Failure isolation** — for random batches with injected failures, assert each job reaches a terminal state and a failed job never changes another job's terminal status
    - **Validates: Requirements 8.1, 8.2, 8.4**

  - [ ]* 20.5 Write property test for visuals never upgrade fit (P2)
    - **Property 2: Visuals never upgrade fit** — for random results and arbitrary `Visualization.generationStatus` values, assert `fitStatus` is invariant under visualization state
    - **Validates: Requirements 8.10, 12.2**

- [~] 21. Implement visualization status polling and gallery integration
  - Add `GET /api/visualizations?searchId` returning per-product `{ status, imageRef? }`; wire `POST /api/search` to enqueue one job per eligible product and return `vizStatus: pending[]`
  - In the gallery/detail views, poll status, show generation status indicators and skeletons, render the generated-image gallery per card, display cards without an in-room image when unavailable, label every visualization as an approximate representation, and expose per-product cancel/retry controls
  - _Requirements: 8.9, 9.2, 9.6, 12.5, 13.6_

- [~] 22. Checkpoint — Phase 3 visualization
  - Ensure all tests pass, ask the user if questions arise.

### Phase 4 — Refinement

- [~] 23. Implement product comparison
  - Build a comparison view across dimensions, price, material, color, rating, clearance, FitStatus, and visualizations for two or more selected products; reject a comparison started with fewer than two products and prompt to select at least two
  - _Requirements: 10.1, 10.2_

- [ ] 24. Implement favorites, saved searches, and optional accounts
  - [~] 24.1 Implement Session_Service persistence
    - Implement `SessionService`: create session, save/list session-scoped favorites, and return previously computed results on revisit; allow the full journey without an account
    - _Requirements: 11.1, 11.2, 11.3_

  - [~] 24.2 Implement optional accounts and deletion
    - Where accounts are enabled, associate saved photos, favorites, and projects with the authenticated user and delete a requested photo or project on request
    - _Requirements: 11.4, 11.5_

  - [ ]* 24.3 Write unit tests for favorites and deletion
    - Test session-scoped favorite persistence/revisit and authenticated deletion of a photo/project
    - _Requirements: 11.1, 11.2, 11.5_

- [~] 25. Implement performance caching and signed-URL TTL bounds
  - Cache generated images keyed by `(roomImageRef, productImageRef)` where provider policy permits, avoiding regeneration on re-sort/revisit; confirm signed storage URLs use short TTLs (≤300s)
  - Add results sort options that reuse cached visualizations without regenerating
  - _Requirements: 8.8, 9.4, 1.10_

- [ ]* 26. Write end-to-end integration tests in DEMO mode
  - Exercise upload → analyze → search → enqueue → poll visualizations → results against DEMO providers; verify mode banners surface, a deliberately failing demo image job isolates from siblings, and no credential appears in any client-bound response
  - _Requirements: 6.3, 8.4, 13.1, 13.4, 13.6_

- [~] 27. Add `.env.example` and setup/deployment documentation
  - Create `.env.example` enumerating all `COMMERCE_*`, `VISION_*`, `IMAGEGEN_*`, database, and storage variables with no secret values, documenting that unset integration vars trigger automatic DEMO mode
  - Write setup/deployment docs covering local dev, running in DEMO mode, supplying live credentials, database/storage provisioning, and the background worker
  - _Requirements: 6.2, 13.4_

- [~] 28. Final checkpoint — all phases
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional (tests) and can be skipped for a faster MVP; core implementation tasks are never optional.
- Each task references specific requirement sub-clauses for traceability.
- Property tests encode the design's correctness properties: P1 (14.3), P2 (20.5), P3 (20.4), P4 (13.2), P5 (7.2), P6 (8.3), P7 (2.5), P8 (2.4). Each is its own sub-task placed near the code it validates.
- Required automated tests from the deliverable are covered: dimension filtering and budget filtering (14.4), unit conversion (7.2, 7.3), product-link handling (15.2), and failure scenarios (5.4, 12.2 via 14.4, 18.3, 26).
- Each phase boots and runs end-to-end in DEMO mode with no external credentials, satisfying the modular, independently testable requirement.
- Checkpoints (tasks 11, 17, 22, 28) mark phase boundaries for incremental validation.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2.1", "7.1"] },
    { "id": 2, "tasks": ["2.2", "7.2", "7.3", "5.1"] },
    { "id": 3, "tasks": ["2.3", "5.2", "8.1"] },
    { "id": 4, "tasks": ["2.4", "2.5", "3", "6", "8.2", "8.3"] },
    { "id": 5, "tasks": ["4", "5.3", "9"] },
    { "id": 6, "tasks": ["5.4", "10.1"] },
    { "id": 7, "tasks": ["10.2", "12.1"] },
    { "id": 8, "tasks": ["12.2", "13.1", "15.1"] },
    { "id": 9, "tasks": ["13.2", "14.1", "15.2"] },
    { "id": 10, "tasks": ["14.2"] },
    { "id": 11, "tasks": ["14.3", "14.4", "16"] },
    { "id": 12, "tasks": ["18.1", "19"] },
    { "id": 13, "tasks": ["18.2", "18.3", "20.1"] },
    { "id": 14, "tasks": ["20.2"] },
    { "id": 15, "tasks": ["20.3", "20.4", "20.5"] },
    { "id": 16, "tasks": ["21"] },
    { "id": 17, "tasks": ["23", "24.1", "25"] },
    { "id": 18, "tasks": ["24.2"] },
    { "id": 19, "tasks": ["24.3", "26", "27"] }
  ]
}
```
