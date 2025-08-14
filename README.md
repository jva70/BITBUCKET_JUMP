# Jump-Repo Cookbook (SAP CI/CD ↔ GitHub ↔ Corporate Bitbucket)

## Why this works

**Problem:** SAP CI/CD (Cloud Foundry Env 2.0 pipeline) can fetch a "source repository" via basic auth only, but VW corporate Bitbucket won't allow basic auth.

**Idea:** Point SAP CI/CD at a small GitHub repo (jump repo) it can clone. In the Build / runFirst hook (which SAP runs in a Node 22 toolbox container), you:

1. `git clone` the real Bitbucket repo using a Bearer HTTP access token,
2. copy the project into the CI workspace,
3. write an overlay config (`dynamicConfig.yml`) to respect the pipeline from the project,
4. stash everything in `cloudcitransfer/` so later stages can reuse it.

Build then runs with `mtaBuild.path: cloudcitransfer`, so it finds your `mta.yaml` from the Bitbucket project and builds your MTAR.

## What you'll build

- **GitHub repo:** `BITBUCKET_JUMP` (contains only `.pipeline/config.yml`).
- **Corporate Bitbucket repo:** your real app (`ibantest`), with a CI params file `.ci/params.yml`.
- **SAP CI/CD job:** `IBANTEST_WORKAROUND`, Source Repository mode, pointing to the GitHub repo, plus one Secret Text credential holding the Bitbucket HTTP access token.

## Prerequisites

- Permission to create repos in GitHub (or equivalent).
- Access to corporate Bitbucket and ability to create HTTP access tokens.
- Access to SAP CI/CD (service instance) and permission to create jobs and credentials.
- A CF account/service key for deployment (optional if you want Acceptance deploy).

## 1) GitHub: create the jump repo

1. Create a private repo named `BITBUCKET_JUMP`.
2. Clone it locally (VS Code is fine).
3. Create the path `.pipeline/config.yml` with the content below and commit/push.

### `.pipeline/config.yml` (works with your logs & screenshots):

```yaml
service:
  buildToolVersion: "MBTJ21N22"   # uses devxci/mbtci-java21-node22

stages:
  Build:
    runFirst:
      # POSIX-/dash-safe script (no bash-only shopt)
      command: |
        set -e
        echo "== Bootstrap: start =="
        if [ -z "${BITBUCKET_TOKEN}" ]; then
          echo "ERROR: BITBUCKET_TOKEN is not set (map a Secret Text credential)."
          exit 1
        fi
        # Send Authorization: Bearer <token> to your Bitbucket host
        git config --global http.https://devstack.vwgroup.com/.extraheader "Authorization: Bearer ${BITBUCKET_TOKEN}"
        echo "[1/3] Cloning corporate repo into ./source ..."
        rm -rf source
        git clone https://devstack.vwgroup.com/bitbucket/scm/vwskdevops/ibantest.git source
        echo "[2/3] Writing pipeline overlay dynamicConfig.yml from Bitbucket repo ..."
        if [ -f source/.ci/params.yml ]; then
          cp source/.ci/params.yml dynamicConfig.yml
          echo "dynamicConfig.yml written from source/.ci/params.yml"
        elif [ -f source/.pipeline/config.yml ]; then
          cp source/.pipeline/config.yml dynamicConfig.yml
          echo "dynamicConfig.yml written from source/.pipeline/config.yml"
        else
          echo "No overlay file found; continuing without dynamicConfig.yml"
        fi
        echo "[3/3] Syncing Bitbucket sources into workspace root so build steps see mta.yaml ..."
        # dash doesn't support 'shopt', so copy explicitly (or use rsync if available)
        cp -a source/README.md source/ibansvc source/mta.yaml source/oauth-generator-script.sh \
              source/package-lock.json source/package.json source/xs-security.json source/xs-security.original ./
        # Keep a copy for later stages (SAP CI/CD restashes this)
        mkdir -p cloudcitransfer
        cp -a source/. cloudcitransfer/.
        echo "== Bootstrap: done =="

# Tell piper where the real sources live after runFirst
steps:
  mtaBuild:
    path: "cloudcitransfer"

# Optional: turn on Acceptance deploy via pipeline (see section 6)
# stages:
#   Acceptance:
#     cloudFoundryDeploy: true
# steps:
#   cloudFoundryDeploy:
#     appName: "ibanTEST"
#     deployType: "standard"   # or 'blue-green'
#     # You can also add cfApiEndpoint/cfOrg/cfSpace here if not provided via UI
```

### Notes

- That script is POSIX sh safe (your logs showed `/bin/sh` → `shopt: not found`—we removed shopt).
- `mtaBuild.path: cloudcitransfer` is required so the build reads `cloudcitransfer/mta.yaml` produced by the bootstrap.

## 2) Corporate Bitbucket: create a HTTP Access Token

1. Open your repo → **Settings** → **HTTP access tokens**.
2. Create new token:
   - **Name:** `IBANTEST_WRITE_[YOUR_ID]` (e.g., `IBANTEST_WRITE_JVANOVCAN`)
   - **Permissions:** Repository: Write
3. Copy the token (example shape: `BBDC-NzE1MzExOTM5ODxxxxxxxxxx8CWAswbD2Aqy6XPJ4M7C`).
4. Save it safely—you won't see it again.

## 3) In the Bitbucket project: add `.ci/params.yml`

If you can, generate it with the CI/CD Job Designer GUI and export to YAML; otherwise create it manually:

### `source/.ci/params.yml` (example you posted):

```yaml
Acceptance:
  cfApiEndpoint: "https://api.cf.eu10-004.hana.ondemand.com"
  cfOrg: "VW_SK_klientportal-tst"
  cfSpace: "DEV_Build_SPC"
  deployType: "standard"
  cloudFoundryDeploy: true
  npmExecuteEndToEndTests: false
```

### What from `.ci/params.yml` is used?

In the setup you ran, build succeeded but no deploy happened. That's because the SAP pipeline actually read its stage switches from `.pipeline/config.yml` (the jump repo), not from `dynamicConfig.yml`.

We copied `.ci/params.yml` to `dynamicConfig.yml` for future use, but by default piper doesn't consume `dynamicConfig.yml` automatically. To deploy, you must either:

- enable `Acceptance` → `cloudFoundryDeploy: true` in the jump repo's `.pipeline/config.yml`, or
- enable/override the stage in the SAP CI/CD Job UI (Stages tab) if you're not using pure "Source Repository" overrides.

(We show how in section 6.)

## 4) SAP CI/CD: create credentials & repository

Create credentials/screens exactly as in your screenshots:

### A) GitHub basic auth (for the jump repo)
- **Name:** `jva`
- **Type:** Username/Password (or a token mapped as Basic).

### B) Bitbucket bearer token
- **Name:** `bitbucket-token`
- **Type:** Secret Text
- **Secret:** paste your Bitbucket HTTP access token.

(This cannot be used as a repository credential, that's OK—your runFirst script reads it as an env var.)

### C) (Optional) Cloud Foundry login for deployment
Create a credential your environment expects (e.g., service key or username/password), e.g. `sap-demo-admin`.

### D) Register the GitHub repository in SAP CI/CD

Use exactly these settings (matching your screenshot):

- **Name:** `IBANTEST_WORKAROUND`
- **Description:** Build of IBANTEST using a jump repo
- **Repository:** `BITBUCKET_JUMP`
- **Branch:** `main`
- **Pipeline:** Cloud Foundry Environment 2.0
- **Configuration Mode:** Source Repository

Register the repository:
- **Name:** `BITBUCKET_JUMP`
- **Clone URL:** `https://github.com/jva70/BITBUCKET_JUMP.git`
- **Credential:** `jva` (Basic)

And the Bitbucket token credential:
- **Name:** `bitbucket-token`
- **Type:** Secret Text
- **Value:** your HTTP access token

We'll map this token into the Build/runFirst step via Additional Commands → the SAP library injects it as `BITBUCKET_TOKEN`.

## 5) How the "runFirst in Build" actually runs

1. SAP CI/CD launches a Node 22 toolbox container (you can see `NODE_VERSION=22.18.0` in logs).
2. Your `.pipeline/config.yml` → `stages.Build.runFirst.command` runs before the normal build steps.
3. We:
   - set a git extra header so requests to `https://devstack.vwgroup.com` include `Authorization: Bearer $BITBUCKET_TOKEN`,
   - clone the corporate repo into `./source`,
   - copy the project into the workspace root and `cloudcitransfer/`,
   - later steps read from `cloudcitransfer/`.

The pipeline step `mtaBuild.path: cloudcitransfer` makes Piper call `mbt build` with `--source cloudcitransfer`, which fixed the earlier "mta.yaml not found" error.

You can see the effect in the successful build log (excerpt):

```
info  mtaBuild - "cloudcitransfer/mta.yaml" file found
info  mtaBuild - Executing mta build call: "mbt build --mtar ibanTEST.mtar --platform CF --source cloudcitransfer ..."
...
INFO the MTA archive generated at: .../cloudcitransfer/ibanTEST.mtar
```

## 6) (Optional) Turn on Acceptance deploy (Cloud Foundry)

Right now the job builds only. To deploy the MTAR in Acceptance:

### Option A — enable in the jump repo (preferred for "Source Repository" mode):

```yaml
# in BITBUCKET_JUMP/.pipeline/config.yml
stages:
  Acceptance:
    cloudFoundryDeploy: true
    npmExecuteEndToEndTests: false   # keep your setting

steps:
  cloudFoundryDeploy:
    deployType: "standard"
    # If your SAP CI/CD instance uses UI-bound CF target, you can omit the next 3:
    cfApiEndpoint: "https://api.cf.eu10-004.hana.ondemand.com"
    cfOrg: "VW_SK_klientportal-tst"
    cfSpace: "DEV_Build_SPC"
    # Provide a credential id if your setup requires it, e.g.:
    # cloudFoundry:
    #   credentialsId: "sap-demo-admin"
```

### Option B — enable in the Job UI (if not fully controlled by Source Repository mode):

Job → Stages → Acceptance → toggle Cloud Foundry Deploy on, set target (API, Org, Space) and choose your CF credential (e.g., `sap-demo-admin`).

### Why your earlier Acceptance block didn't deploy:

The values were present in `.ci/params.yml` inside the Bitbucket repo, but the pipeline's effective configuration came from the jump repo's `.pipeline/config.yml` (Source Repository mode). Since we didn't turn on `stages.Acceptance.cloudFoundryDeploy: true` there, SAP CI/CD skipped Acceptance ("skipped due to when conditional" in your log).

## 7) Run the job

1. Trigger `IBANTEST_WORKAROUND`.
2. Watch for three key markers in the logs:
   - `Executing code in toolbox image: 'node@sha256:...'`
   - Cloning `... ibantest.git` and listing of `cloudcitransfer`
   - `mbt build --source cloudcitransfer` and final `ibanTEST.mtar` path
3. If Acceptance deploy is enabled, the stage will run after Build and push the MTAR.

## 8) Troubleshooting tips

- **`shopt: not found`** → you're in `/bin/sh` (dash), not bash. Use POSIX-safe copy (as in the cookbook) or `rsync -a source/ ./` if rsync is available in the toolbox.
- **`mta.yaml not found`** → ensure `steps.mtaBuild.path: cloudcitransfer` (exactly as shown) and your runFirst actually copied the repo.
- **Bitbucket auth fails** → confirm the `bitbucket-token` exists and is Secret Text, and the domain in `git config --global http.https://<host>/.extraheader` exactly matches your repo URL origin.
- **Acceptance skipped** → enable `stages.Acceptance.cloudFoundryDeploy: true` in the jump repo `.pipeline/config.yml` (or via the Job UI).

## 9) Minimal files recap

### GitHub `BITBUCKET_JUMP`:
```
.pipeline/
└─ config.yml   # from this cookbook
README.md       # optional
```

### Corporate Bitbucket `ibantest`:
```
.ci/
└─ params.yml   # Acceptance values (optional unless you wire it in)
mta.yaml
ibansvc/...
package*.json, etc.
```

## 10) Security notes

- The Bitbucket HTTP access token never leaves SAP CI/CD's vault; it's injected into the runFirst container as `BITBUCKET_TOKEN`.
- The token is used only to clone the Bitbucket repo from the toolbox container. Nothing is pushed back unless you add such steps.
- Avoid echoing the token; SAP masks it, but don't print environment variables wholesale.




# BITBUCKET_JUMP

In GitHub
- create a new repo in gitHub (BITBUCKET_JUMP)
- clone the repo into dev environment (VSC)
- create .pipeline/config.yml, commit

In devstack/corporate Bitbucket
- Generate HTTP Access Token in BitBucket
- 1. Navigate to Repository Settings

-- Go to your repository → Settings
-- Select HTTP access tokens
- 2. Create New Token

-- Name: IBANTEST_WRITE_[YOUR_ID] (e.g., IBANTEST_WRITE_JVANOVCAN)
-- Permissions: Repository Write
-- Copy the generated token (e.g., BBDC-NzE1MzExOTM5ODxxxxxxxxxx8CWAswbD2Aqy6XPJ4M7C)
-- ⚠️ Important: Save this token securely - it won't be shown again!


In the SAP project (in BAS tool) that is stored in corporate Bitbucket
- create .ci/params.yml file with the content below (ideally create one with GUI job designer and then export it as YAML)


In SAP CI/D create
- credentials to connect to BITBUCKET_JUMP (jva/Basic authentication)
- BITBUCKET_JUMP repository (name BITBUCKET_JUMP, URL: https://github.com/jva70/BITBUCKET_JUMP.git, credentials = jva, remove webhook cfg)
- IBANTEST_WORKAROUND job (Description: Build of IBANTEST using a jump repo, Repository: BITBUCKET_JUMP, Branch: main, Pipeline: Cloud Foundry Environment 2.0
Configuration Mode: Source Repository)
- Credential bitbucket-token, Type: secret text, value BBDC-NzE1MzExOTM5ODxxxxxxxxxx8CWAswbD2Aqy6XPJ4M7C
- Credentials to log in back to CloudFoundry (sap-demo-admin)

Run the job

