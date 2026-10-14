# Preview validation record

Status: **unreleased**. This record describes checks on the development branch, not a production readiness certification. Customer configuration and raw Azure evidence remain outside Git.

## Verified on 2026-10-14

- Python suite: 59 passing tests, including token verification, role boundaries, suspension, revision conflicts, import validation, policy compilation, and publication rollback behavior.
- Frontend: strict TypeScript production build and eight CSV/estimate/authentication tests pass. Authentication tests cover rejected redirects without automatic retries, cached-account reuse, the absence of a forced account picker, and failed bootstrap handling.
- Infrastructure: Bicep compiles; the isolated Azure Functions preview package deploys with managed identity and private Blob networking.
- Hosted public endpoints: health and sign-in bootstrap return 200; unauthenticated state access returns 401.
- Separate APIM preview API: all nine deployed Anthropic models answer small real inference requests. Streaming, token counting, missing/invalid credentials, model denial, filtered catalog, suspension, and an organization token quota were checked. Original preview policies were restored afterward.
- Application Insights: live preview request aggregates and current-month token metrics load using local administrator credentials. No prompt/response capture is enabled.
- Scale of configuration: 2,000-member CSV import and generated policy size pass automated checks. This is not a concurrency or throughput test.
- UI: overview, live usage/budgets, group editor, command search with Enter navigation, dialog focus restoration, and a 390-pixel layout were inspected in the browser.
- The existing production gateway policy was compared with its pre-test hash and remains unchanged.

Hosted sign-in is still pending. In the observed sign-in sequence, Entra returned `AADSTS50012`; an available app-scoped log recorded `130500` with phone sign-in blocked by User Credential Policy, despite the MFA requirement being satisfied. The portal now retains its sign-in screen with diagnostic details after a rejected redirect. No authentication-method or tenant policy was changed.

## Still required before release

- Complete the hosted owner's Entra sign-in, then verify private Blob state read/write, Insights access, and APIM publication through the portal's managed identity.
- Exercise real administrator, billing, auditor, and developer sessions under customer Conditional Access and consent policies.
- Validate each quota/rate scope, publication interruption and recovery, two-administrator conflicts, and Blob version recovery against Azure.
- Run the full Mac onboarding journey from an exported portal profile and verify model defaults and revocations.
- Complete keyboard/screen-reader and error-state coverage, operational monitoring, retention and recovery procedures, security review, and load/cost measurements for the intended deployment size.

The detailed release checklist and cost limitations are in [README.md](README.md). Do not merge into `main`, tag, or distribute a customer release until those checks are complete and release is explicitly approved.
