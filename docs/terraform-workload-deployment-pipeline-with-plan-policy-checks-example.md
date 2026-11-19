# Deploying a Terraform Workload With Plan Policy Checks

## Pipelines Running on Unprotected Branches

```mermaid
flowchart TB
  start
  plan-test-platx-global["Plan - Test Platform X Global"]
  plan-prod-platx-global["Plan - Prod Platform X Global"]
  fmt-check["Format Check"]
  pc-test-platx-global["Plan Policy Check - Test Platform X Global"]
  pc-prod-platx-global["Plan Policy Check - Prod Platform X Global"]
  success
  start --> fmt-check --> success
  start --> plan-test-platx-global & plan-prod-platx-global
  plan-test-platx-global --> pc-test-platx-global
  plan-prod-platx-global --> pc-prod-platx-global
  pc-prod-platx-global & pc-test-platx-global --> success
```

## Pipelines Running on Protected Branches

```mermaid
flowchart TB
  start
    plan-test-platx-global["Plan - Test Platform X Global"]
    pc-test-platx-global["Plan Policy Check - Test Platform X Global"]
    apply-test-platx-global["Apply Test Platform X Global"]
    test-test["Tests"]
    plan-prod-platx-global["Plan - Prod Platform X Global"]
    pc-prod-platx-global["Plan Policy Check - Prod Platform X Global"]
    apply-prod-platx-global["Apply Prod Platform X Global"]
  success

  start --> plan-test-platx-global
  plan-test-platx-global --> pc-test-platx-global --> apply-test-platx-global --> test-test
  test-test --> plan-prod-platx-global
  plan-prod-platx-global --> pc-prod-platx-global --> apply-prod-platx-global --> success
```

## Example Implementation

```yml
include:
  # Deployment Workflow Components
  - component: ${CI_SERVER_FQDN}/engineering/dev-tools/internal/gitlab-cicd-deployment-workflow/workflow@2
  - component: ${CI_SERVER_FQDN}/engineering/dev-tools/internal/gitlab-cicd-deployment-workflow/deploy-if-protected@2
    inputs:
      environment-deployment-tier: testing
      environment-name: test/platx/global
  - component: ${CI_SERVER_FQDN}/engineering/dev-tools/internal/gitlab-cicd-deployment-workflow/deploy-if-protected@2
    inputs:
      environment-deployment-tier: production
      environment-name: prod/platx/global
  # Terraform Components
  - component: ${CI_SERVER_FQDN}/engineering/dev-tools/internal/gitlab-cicd-terraform/format-check@4
    inputs:
      terraform-image-version: "1.6"
  - component: ${CI_SERVER_FQDN}/engineering/dev-tools/internal/gitlab-cicd-terraform/plan-and-apply@4
    inputs:
      environment: test
      provider: platx
      region: global
  - component: ${CI_SERVER_FQDN}/engineering/dev-tools/internal/gitlab-cicd-terraform/plan-and-apply@4
    inputs:
      environment: prod
      provider: platx
      region: global

stages:
  - main

default:
  image:
    name: ${CI_DEPENDENCY_PROXY_DIRECT_GROUP_IMAGE_PREFIX}/hashicorp/terraform:1.6
    entrypoint:
      - "/usr/bin/env"
      - "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# ---
# Hidden Jobs
# ---
# These hidden jobs that are extended by jobs that run in either protected or unprotected contexts.
#
# It's a way to DRY up the code for:
#
# 1. Retrieving temporary credentials
# 2. Extending the terraform cicd components
# ---

.plan:test:platx:global:
  extends:
    - .plan:test:platx:global
  before_script:
    # Retrieve temporary credentials authorized to:
    #
    # 1. read test tf state
    # 2. read infrastructure in the test platform x environment
    - echo "retrieving temporary credentials...";

.plan:prod:platx:global:
  extends:
    - .plan:prod:platx:global
  before_script:
    # Retrieve temporary credentials authorized to:
    #
    # 1. read prod tf state
    # 2. read infrastructure in the prod platform x environment
    - echo "retrieving temporary credentials...";

.plan-policy-check:test:platx:global:
  script:
    # Performs a policy check against the planned changes to the test platform x environment. Consider OPA.
    - echo "checking plan against policy..."

.plan-policy-check:prod:platx:global:
  script:
    # Performs a policy check against the planned changes to the prod platform x environment. Consider OPA.
    - echo "checking plan against policy..."

# ---
# Unprotected Pipeline
# ---
# These jobs are run when the pipeline context is unprotected (e.g., unprotected branch pipelines).
#
# They extend:
#
# 1. The workflow component hidden job `.run-if-unprotected` to only run the job in unprotected contexts.
# 2. The previously defined hidden jobs such as `.plan:test:platx:global` and `.plan-policy-check:test:platx:global` for DRY profit.
#
# They use the 'needs' keyword to construct the following DAG:
#
# - start --> unprotected:plan:test:platx:global & unprotected:plan:prod:platx:global
# - unprotected:plan:test:platx:global --> unprotected:plan-policy-check:test:platx:global
# - unprotected:plan:prod:platx:global --> unprotected:plan-policy-check:prod:platx:global
# - unprotected:plan-policy-check:test:platx:global & unprotected:plan-policy-check:prod:platx:global --> end
# ---

terraform-format-check:
  stage: main
  extends:
    - .format-check
    - .run-if-unprotected

unprotected:plan:test:platx:global:
  stage: main
  extends:
    - .plan:test:platx:global
    - .run-if-unprotected

unprotected:plan:prod:platx:global:
  stage: main
  extends:
    - .plan:prod:platx:global
    - .run-if-unprotected

unprotected:plan-policy-check:test:platx:global:
  stage: main
  extends:
    - .plan-policy-check:test:platx:global
    - .run-if-unprotected
  needs:
    - unprotected:plan:test:platx:global

unprotected:plan-policy-check:prod:platx:global:
  stage: main
  extends:
    - .plan-policy-check:prod:platx:global
    - .run-if-unprotected
  needs:
    - unprotected:plan:prod:platx:global

# ---
# Protected Pipeline
# ---
# These jobs are run when the pipeline context is protected (e.g., protected branch pipelines).
#
# They extend:
#
# 1. The deployment workflow component hidden job `.run-if-protected` to only run the job in protected contexts.
# 2. The deployment workflow component hidden jobs such as `.deploy-if-protected:testing:test/platx/global` to add deployment context to the jobs.
# 3. The previously defined hidden jobs such as `.plan:test:platx:global` and `.plan-policy-check:test:platx:global` for DRY profit.
#
# They use the 'needs' keyword to construct the following DAG:
#
# - start --> protected:plan:test:platx:global
# - protected:plan:test:platx:global --> protected:plan-policy-check:test:platx:global
# - protected:plan-policy-check:test:platx:global --> apply:test:platx:global
# - apply:test:platx:global --> test:test
# - test:test --> protected:plan:prod:platx:global
# - protected:plan:prod:platx:global --> protected:plan-policy-check:prod:platx:global
# - protected:plan-policy-check:prod:platx:global --> apply:prod:platx:global
# - apply:prod:platx:global --> end
# ---

protected:plan:test:platx:global:
  stage: main
  extends:
    - .plan:test:platx:global
    - .run-if-protected

protected:plan-policy-check:test:platx:global:
  stage: main
  extends:
    - .plan-policy-check:test:platx:global
    - .run-if-protected
  needs:
    - protected:plan:test:platx:global

apply:test:platx:global:
  stage: main
  extends:
    - .apply:test:platx:global
    - .deploy-if-protected:testing:test/platx/global
  needs:
    - protected:plan:test:platx:global
    - protected:plan-policy-check:test:platx:global
  before_script:
    # Retrieve temporary credentials authorized to:
    #
    # 1. manage test tf state
    # 2. manage infrastructure in the test platform x environment
    - echo "applying changes ..."

test:test:
  stage: main
  needs:
    - apply:test:platx:global
  script:
    # Runs integration tests against the deployment to the test platform x environment
    - echo "running integration tests ..."

protected:plan:prod:platx:global:
  stage: main
  extends:
    - .plan:prod:platx:global
    - .run-if-protected
  needs:
    - test:test

protected:plan-policy-check:prod:platx:global:
  stage: main
  extends:
    - .plan-policy-check:prod:platx:global
    - .run-if-protected
  needs:
    - protected:plan:prod:platx:global

apply:prod:platx:global:
  stage: main
  extends:
    - .apply:prod:platx:global
    - .deploy-if-protected:production:prod/platx/global
  needs:
    - protected:plan:prod:platx:global
    - protected:plan-policy-check:prod:platx:global
  before_script:
    # Retrieve temporary credentials authorized to:
    #
    # 1. manage prod tf state
    # 2. manage infrastructure in the prod platform x environment
    - echo "applying changes ..."
```

### Credential Management Considerations

**Important note for users:**

This pipeline example **does not include credential retrieval or environment access logic.** Instead, it provides `before_script` placeholders in each plan and apply job for inserting credential retrieval commands.

Users are encouraged to integrate a solution like gitlab-cicd-temp-creds components to automate the retrieval and injection of **temporary access credentials** needed for interacting with Terraform backends and cloud providers.
