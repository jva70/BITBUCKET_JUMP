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


# source/.ci/params.yml  (in corporate repo)
general:
  buildTool: "mta"
service:
  buildToolVersion: "MBTJ21N22"
  stages:
    Acceptance:
      # the name of the credential stored in SAP CI/CD credential store
      cfCredentialsId: "sap-demo-admin"
stages:
  Build:
    mavenExecuteStaticCodeChecks: false
    npmExecuteLint: false
  Acceptance:
    cfApiEndpoint: "https://api.cf.eu10-004.hana.ondemand.com"
    cfOrg: "VW_SK_klientportal-tst"
    cfSpace: "DEV_Build_SPC"
    deployType: "standard"
    cloudFoundryDeploy: true
    npmExecuteEndToEndTests: false
  Malware Scan:
    malwareExecuteScan: false
  Release:
    tmsExport: false
    tmsUpload: false
    cloudFoundryDeploy: false
  Additional Unit Tests:
    npmExecuteScripts: false
  Compliance:
    sonarExecuteScan: false
steps:
  cloudFoundryDeploy:
    mtaDeployParameters: "-f --version-rule ALL"
  artifactPrepareVersion:
    versioningType: "cloud_noTag"    