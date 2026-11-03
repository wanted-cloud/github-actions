# wanted-cloud/github-actions

Shared GitHub Actions workflows for the WANTED.solutions estate.

Public **because it has to be**: GitHub only allows a reusable workflow to be called
from another organisation if the repository holding it is public. It is called from
`mrlm-net`, `creative-pioneers`, `mycrew-online` and `wanted-cloud`. Nothing here
holds a secret — credentials come from the caller's organisation secrets.

## `sbom.yml` — SBOM generation and Dependency-Track upload

Generates a CycloneDX SBOM and uploads it to Dependency-Track. The Azure DevOps
equivalent lives in `infrastructure/vsts-pipeline-templates` (`templates/cdx/*`); this
is the same behaviour for repositories built by Actions, and the two are meant to
produce the same component counts for the same repository.

```yaml
jobs:
  sbom:
    uses: wanted-cloud/github-actions/.github/workflows/sbom.yml@v1
    with:
      sbom-type: npm
      dt-project: mycrew-website
      dt-parent: websites
      dt-tags: tier=website,language=npm
      pnpm-filter: "@mycrew/website"
    secrets: inherit
```

### Inputs

| input | required | default | notes |
|---|---|---|---|
| `sbom-type` | yes | — | `golang`, `npm`, `yarn`, `php` |
| `dt-project` | yes | — | project name in Dependency-Track |
| `dt-parent` | no | `""` | parent container; must already exist in DT |
| `dt-tags` | no | `""` | comma-separated, e.g. `tier=website,language=npm` |
| `go-main` | for `golang` | — | e.g. `./cmd/app` — scopes the SBOM to one binary |
| `pnpm-filter` | for `npm` | — | workspace package name |
| `php-dir` / `php-flavour` | for `php` | `.` / `composer` | `composer`, `php`, `wordpress` |
| `min-third-party` | no | `10` | floor; below this the upload is refused |

### Secrets

Set as **organisation** secrets, one DT key per org:

- `DEPTRK_API_KEY` — a `ci-sbom` team key
- `CF_ACCESS_CLIENT_ID` / `CF_ACCESS_CLIENT_SECRET` — Cloudflare Access service token

`security.outlays.ws` is behind Cloudflare Access. Without the service token the
upload follows a 302 into an HTML login page, and the failure reads like a rejected
API key rather than a missing one.

### The project must exist first

Declare it in `deptrk/projects.yaml` **and** `deptrk/integrations.yaml` in the
`production-contributors` repository. An undeclared component makes the reconciler
fail — deliberately, so a project cannot end up tracked in one place and invisible in
the other.

### Why the build fails on `REFUSING TO PUBLISH`

That is the guard working. A collector that silently does nothing still emits valid
CycloneDX with zero components, and that uploads cleanly and reports a project with no
vulnerabilities. Refusing to publish is the only thing separating "nothing is wrong"
from "nothing was scanned". Check that dependency installation actually ran.
