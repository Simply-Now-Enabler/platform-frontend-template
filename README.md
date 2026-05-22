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

The `build` script in `package.json` is `tsc --noEmit && vite build` by design. The `&&` chain is load-bearing: a TypeScript error must fail the entire script with a non-zero exit code so the SWA GitHub Actions deploy workflow sees a failed build step and withholds deployment. Without the `--noEmit` guard, `tsc` exits zero even when type errors are present (it still emits JS), and Oryx — the Azure Static Web Apps build agent — ships whatever stale `dist/` content remains from a prior successful build. This silent-fallback bug was surfaced on 2026-05-14 and fixed in platform build 03-C-4-e; this commit re-lands the fix in the template repo (the original fix mistakenly landed in the platform repo's `packages/frontend/` copy instead). Do not replace this script with a tolerant pattern such as `vite build` alone or any wrapper that swallows exit codes — doing so silently breaks the deploy safety guarantee. Client repos provisioned from this template inherit the script at provision time; preserving the exit-code behaviour in downstream repos is the operator's responsibility. The workflow at `.github/workflows/azure-static-web-apps.yml` invokes this script via the SWA action's `app_build_command` parameter; removing that parameter would cause the SWA action's Oryx to bypass this script and silently build using its own framework detection — re-introducing the Oryx silent-fallback failure mode.
