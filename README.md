# Performance Enabler frontend template

This repository is a **GitHub template** for Performance Enabler Digital Platform client apps. Do not commit changes here directly — clone it, customise locally, and use the **"Use this template"** button to generate per-client repositories.

Repos generated from this template are populated by the platform's `provisionAppIntegrated` Function (or by the self-service provisioning script for pull-request / export-only modes). The CI workflow expects six GitHub Action secrets to be set on the generated repository:

| Secret | Source |
|---|---|
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | Auto-set by `az staticwebapp create` (or by `provisionAppIntegrated`) |
| `VITE_DATAVERSE_URL` | The client's app-data Dataverse URL |
| `VITE_TENANT_ID` | The client's Entra tenant ID |
| `VITE_SP_CLIENT_ID` | The client's published-app Entra App Registration client ID |
| `VITE_REDIRECT_URI` | The SWA's `https://*.azurestaticapps.net` URL |
| `VITE_APP_SLUG` | The app's slug (matches a `*.app.json` file in `public/`) |

For more, see [platform docs](https://github.com/Simply-Now-Enabler/) (operator-internal).

## Build script — load-bearing exit code

The workflow at `.github/workflows/azure-static-web-apps.yml` runs an explicit `npm ci && npm run build` step (via `actions/setup-node@v4`) **before** the `Azure/static-web-apps-deploy@v1` action. The five `VITE_*` build-time env vars are injected on this step — not on the SWA action — so Vite inlines their values during `npm run build`. The SWA action is configured with `skip_app_build: true` and `app_location: "dist"` so Oryx is not involved in the build at all: it zips the pre-built `dist/` directory directly and uploads it.

The `build` script in `package.json` is `tsc --noEmit && vite build`: if `tsc --noEmit` encounters any TypeScript error, the script exits non-zero, the "Install and build" workflow step fails red, and the SWA action never runs. No stale or broken content is deployed.

An earlier iteration placed the `env:` block on the SWA action rather than on the build step. Vite does not see the SWA action's `env:` — `npm run build` runs in a separate step with no `VITE_*` vars in scope, so every `import.meta.env.VITE_*` is inlined as `undefined`. The SWA deployed a build that passed TypeScript but produced a blank-screen SPA with MSAL initialising as `clientId: undefined`. Confirmed live 2026-05-24. Moving the `env:` block to the "Install and build" step resolves this.

Do not remove the "Install and build" step, remove `skip_app_build: true`, move the `env:` block back to the SWA action, or replace the build script with a tolerant pattern such as `vite build` alone — any of these re-introduces a silent or undefined-config failure mode. Client repos provisioned from this template inherit the workflow and script at provision time; preserving this behaviour in downstream repos is the operator's responsibility.
