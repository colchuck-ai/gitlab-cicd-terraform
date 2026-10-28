# Terraform Module Release Pipeline Example

## Example `.gitlab-ci.yml`

```yml
# Note: $CI_PROJECT_NAME is tf-some-provider-my-module

include:
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-release-workflow/release-workflow@1
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-terraform/format-check@3
    inputs:
      stage: test
      terraform-image-version: "1.6"
  - component: gitlab.example.com/engineering/dev-tools/internal/gitlab-cicd-terraform/publish-module@3
    inputs:
      terraform-module-system: "some-provider"

stages:
  - test
  - release
  - publish

test:
  stage: test
  script:
    - echo "tests passed!"

release:
  image: ${CI_DEPENDENCY_PROXY_DIRECT_GROUP_IMAGE_PREFIX}/node:21
  stage: release
  extends:
    - .release-rules
  script:
    - npm install
    - npx semantic-release

publish:
  stage: publish
  extends:
    - .publish-rules
    - .publish-module:tf-some-provider-my-module
```
