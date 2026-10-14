# Management portal — unreleased development branch

All portal work belongs on `feat/admin-portal`. Do not merge it into `main`, tag a release, distribute a customer build, or connect it to a production inference API until the release checks below are complete. CI validates this branch; it does not deploy or publish it.

This is a customer-owned control plane for the existing standard APIM gateway. It uses a React interface, an authenticated Python API on Azure Functions Flex Consumption, and private Azure Blob state. The portal is not in the inference request path. Reviewed access policies are compiled and applied to one explicitly configured APIM API.

## Working scope

- Single-tenant Entra sign-in with authorization code and PKCE; server-verified access tokens.
- Owner, administrator, billing, auditor, and developer roles, enforced by the API.
- Member registration by Entra object ID, suspension, primary groups, personal model grants, and CSV import with validation before saving.
- Model access requests, decisions, versioned drafts, publication review, and audit records.
- Organization, group, and developer token quotas; individual request and token rate limits.
- Application Insights request analytics and current-month token metrics without conversation content.
- Model planning rates and dollar estimates, with missing-rate and threshold notices. Dollar budgets do not stop inference or reconcile Azure invoices.
- Client profile export for the existing Mac installer. Supply the separate desktop public-client ID for MSAL onboarding without Azure CLI.

The portal does not replace Entra account lifecycle, Conditional Access, device management, Claude conversation controls, Azure Cost Management, or an immutable compliance archive. It does not claim Anthropic enterprise certification, SCIM support, invoice management, or legal/compliance guarantees.

## Local development

Use a separate preview API and a dedicated local state file. Local mode is an explicit administrator bypass for development; the server rejects non-loopback hosts/origins and refuses to start in Azure with this mode enabled.

```bash
uv sync --frozen --extra auth --extra portal
npm --prefix portal/web ci --ignore-scripts
npm --prefix portal/web run build

PORTAL_LOCAL_MODE=1 \
PORTAL_ORGANIZATION_PATH=.local/portal/organization.json \
PORTAL_OWNER_ID='<your tenant object ID>' \
PORTAL_OWNER_NAME='Organization owner' \
PORTAL_OWNER_EMAIL='owner@example.com' \
PORTAL_LOCAL_PATH=.local/portal/state.json \
uv run --extra portal uvicorn claude_foundry.portal.app:create_app \
  --factory --host 127.0.0.1 --port 8787
```

Open `http://127.0.0.1:8787`. For live analytics set `PORTAL_INSIGHTS_APP_ID` to the Application Insights application ID. The local server uses Azure CLI credentials for the organization’s subscription and tenant. `npm --prefix portal/web run dev` provides frontend hot reload on port 5173 and proxies `/api` to port 8787.

Use the authenticated Azure preview for real Entra sign-in testing. Local administrator mode deliberately has no user-switching shortcut and is not evidence that authentication works.

## Isolated Azure preview

The administrator needs permissions to create resources and scoped role assignments, plus permission to register a single-tenant Entra application. Developers need neither subscription access nor the administrator CLI.

1. Prepare a separate gateway API. This script rejects a route or API ID that does not end in `-preview`, preserves the original API, reuses its diagnostics, and grants the APIM managed identity inference access to the configured Foundry account.

   ```bash
   uv run python scripts/prepare_portal_preview.py \
     --organization .local/organization.json \
     --output .local/portal/organization.json
   ```

2. Register a dedicated portal SPA/API. There is no client secret. Use `--app-id` when updating an existing registration; the script owns that registration’s redirect URIs and exposed scope.

   ```bash
   uv run python scripts/register_portal.py \
     --organization .local/portal/organization.json \
     --name 'Claude Foundry Admin Preview' \
     --redirect-uri 'https://<portal-name>.azurewebsites.net/'
   ```

   Tenant consent and Conditional Access policies still apply. Register the separate desktop public client using the main repository’s administrator guide if users should onboard without Azure CLI. Never use the portal SPA client ID as the desktop client ID.

3. Create a dedicated resource group. Copy `portal/parameters.example.json` to `.local/portal/parameters.json`, fill in the customer values, and replace its `organization` object with the preview organization configuration. Do not commit customer configuration or outputs.

   ```bash
   az group create --subscription '<subscription>' \
     --name '<preview-resource-group>' --location '<supported Flex region>'

   az deployment group create --subscription '<subscription>' \
     --resource-group '<preview-resource-group>' \
     --template-file infra/portal.bicep \
     --parameters @.local/portal/parameters.json
   ```

   The template creates a consumption plan with no always-ready instances, private storage with shared keys disabled, a managed identity, and RBAC limited to one APIM API’s policies and the selected Insights component. Blob versioning and 30-day deletion recovery are enabled. Private storage networking is enabled by default, with a private endpoint and a dedicated VNet. Choose CIDRs compatible with the customer’s network. The template does not enable public access to storage when `privateStorage=true`.

   Create the app with its private network integration from the start. In preview validation, adding private networking after creating an empty Flex app left code deployment unable to reach storage; recreating that empty app with this template resolved it. Do not recreate an existing customer app as a routine repair step.

4. Build and deploy the application package. The package contains an explicit source/static asset allowlist and locked Python requirements; customer state and credentials are excluded.

   ```bash
   uv run python scripts/build_portal.py
   az functionapp deployment source config-zip \
     --subscription '<subscription>' --resource-group '<preview-resource-group>' \
     --name '<portal-name>' --src dist/claude-foundry-admin-preview.zip \
     --build-remote true --timeout 600
   ```

   A successful infrastructure deployment is not proof that code deployment or private networking works. Verify `/api/health`, the sign-in flow, an authenticated state read/write, telemetry, and publication with the app’s managed identity. Treat storage `403` during deployment as an unresolved access/network problem; do not enable shared keys or remove network restrictions to get a green result.

5. Sign in as the configured owner. Register developers, review group grants and token limits, and publish to the preview API. Direct user Foundry RBAC bypasses gateway governance and must be reviewed separately before a production rollout.

## Validation and release checks

```bash
uv run --extra auth --extra portal ruff check src scripts tests
uv run --extra portal ruff format --check src scripts tests
uv run --extra auth --extra portal pytest -q
npm --prefix portal/web run build
npm --prefix portal/web test
az bicep build --file infra/portal.bicep --outfile /tmp/portal.arm.json
```

The live governance test temporarily changes **only a preview API** and restores both policies. It sends small billable inference requests. Supply a state file containing the signed-in test principal; never use a production route.

```bash
uv run --extra portal python scripts/verify_portal_preview.py \
  --organization .local/portal/organization.json \
  --state .local/portal/state.json --all-models
```

Before release, record evidence for all of the following:

- Fresh customer deployment, including private networking, code deployment, Entra consent, and managed identity permissions.
- Real owner/admin/billing/auditor/developer sessions, suspension, expiry, wrong-tenant tokens, and unauthorized requests.
- Model grants/revocations, catalog filtering, streaming, token counting, rate limits, all quota scopes, and failed-publication recovery.
- Large-team import, two-administrator conflict handling, pagination, and recovery from a prior Blob version.
- Usage attribution, sampling, missing metrics, estimate limitations, and customer-approved dollar-budget behavior.
- Keyboard and screen reader checks, light/dark themes, narrow layouts, error/loading/empty states, and the Mac onboarding journey.
- Load/cost measurements at the intended scale, retention and data-residency decisions, security review, and explicit approval to release.

## Cost and operating limits

Functions compute scales to zero; storage, Private Link, private DNS, and monitoring can still incur charges. The instance maximum is a scaling ceiling, not reserved capacity or a dollar spending cap. The default is 40 because that is the Flex platform’s minimum allowed ceiling. Existing APIM and Foundry costs are separate.

Private Blob networking adds a small fixed cost and is required in organizations that disable public storage access. A customer that permits identity-protected public storage endpoints can explicitly set `privateStorage=false`; this is a different network posture and must fit their organization policy. No database server, container registry, Redis, or always-on portal instance is required.

Token quotas are enforced by APIM within a gateway region. Streaming uses token estimates, concurrent requests can overshoot, and different gateway regions do not share one globally atomic counter. This preview is designed around one standard APIM gateway. A 2,000-member policy-size check is not a 2,000-concurrent-developer load test.

Application state and audit events share a versioned Blob with ETag concurrency control. Azure administrators with Blob write access can change that state. A dedicated immutable audit sink, retention automation, emergency publication recovery, and broader multi-region controls require separate review before enterprise production use.
