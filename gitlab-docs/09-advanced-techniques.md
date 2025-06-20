# Advanced CI/CD Techniques

After mastering the basics of GitLab CI/CD, you can leverage more advanced features to create sophisticated, efficient, and powerful pipelines. This document covers advanced techniques to take your CI/CD workflows to the next level.

## Pipeline Optimization

### Parallel Execution

Split large jobs into parallel tasks to speed up your pipeline:

```yaml
test:
  parallel: 5
  script:
    - echo "Running test suite $CI_NODE_INDEX of $CI_NODE_TOTAL"
    - bin/parallel-test --split=$CI_NODE_INDEX/$CI_NODE_TOTAL
```

### Matrix Jobs

Execute the same job with different variable combinations:

```yaml
test-matrix:
  parallel:
    matrix:
      - BROWSER: [firefox, chrome, safari]
        RESOLUTION: [1080p, 4k]
  script:
    - echo "Testing on $BROWSER at $RESOLUTION"
    - npm run test:$BROWSER -- --resolution=$RESOLUTION
```

This creates six jobs, one for each combination of browser and resolution.

### Directed Acyclic Graph (DAG)

Use the `needs` keyword to create complex job dependencies beyond simple stages:

```yaml
stages:
  - build
  - test
  - deploy

build-backend:
  stage: build
  script:
    - echo "Building backend..."
  artifacts:
    paths:
      - backend/

build-frontend:
  stage: build
  script:
    - echo "Building frontend..."
  artifacts:
    paths:
      - frontend/

test-backend:
  stage: test
  needs:
    - build-backend
  script:
    - echo "Testing backend..."

test-frontend:
  stage: test
  needs:
    - build-frontend
  script:
    - echo "Testing frontend..."

deploy-backend:
  stage: deploy
  needs:
    - test-backend
  script:
    - echo "Deploying backend..."

deploy-frontend:
  stage: deploy
  needs:
    - test-frontend
  script:
    - echo "Deploying frontend..."
```

With this configuration, the backend and frontend pipelines can run independently, without waiting for each other.

## Advanced Job Control

### Child Pipelines

Split your CI/CD configuration into multiple files:

```yaml
# Main .gitlab-ci.yml
stages:
  - generate
  - deploy

generate-pipelines:
  stage: generate
  script:
    - echo "Generating pipelines..."
    # Generate dynamic pipeline configuration
    - ./generate-config.sh > generated-config.yml
  artifacts:
    paths:
      - generated-config.yml

trigger-child-pipeline:
  stage: deploy
  trigger:
    include:
      - artifact: generated-config.yml
        job: generate-pipelines
```

This technique enables:
- Dynamic CI/CD configuration
- Breaking down complex pipelines
- Reusing pipeline templates

### Multi-project Pipelines

Trigger pipelines in other projects:

```yaml
trigger-downstream:
  stage: deploy
  trigger:
    project: group/downstream-project
    branch: main
    strategy: depend  # Wait for downstream pipeline to complete
```

## Template Reuse

### Include Templates

Reuse configuration from other files or projects:

```yaml
include:
  # Local file
  - local: '/templates/build-template.yml'
  # Remote file from same project, different branch
  - project: 'my-group/my-project'
    ref: 'main'
    file: '/templates/test-template.yml'
  # Template from GitLab
  - template: 'Jobs/Build.gitlab-ci.yml'
```

### YAML Anchors and References

Use YAML anchors to create reusable configuration blocks:

```yaml
.default-config: &default-config
  image: alpine:latest
  before_script:
    - echo "Preparing environment..."
  after_script:
    - echo "Cleaning up..."

.deploy-script: &deploy-script
  - echo "Deploying application..."
  - rsync -avz ./dist/ $DEPLOY_SERVER:$DEPLOY_PATH

deploy-staging:
  <<: *default-config
  stage: deploy
  script:
    - *deploy-script
  environment:
    name: staging
```

## Advanced Rules and Workflow Control

### Complex Rules

Create sophisticated rules for job execution:

```yaml
deploy-review:
  script:
    - echo "Deploying review app..."
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - "src/**/*"
      when: manual
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: always
    - when: never
```

### Workflow Rules

Control when pipelines run at a global level:

```yaml
workflow:
  rules:
    - if: '$CI_COMMIT_MESSAGE =~ /skip-ci/'
      when: never
    - if: '$CI_COMMIT_TAG'
      when: always
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - when: never  # Skip pipelines for all other cases
```

## Infrastructure as Code

### Terraform Integration

Manage infrastructure with Terraform:

```yaml
terraform-plan:
  image: hashicorp/terraform:latest
  script:
    - terraform init
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - plan.tfplan

terraform-apply:
  image: hashicorp/terraform:latest
  script:
    - terraform init
    - terraform apply plan.tfplan
  dependencies:
    - terraform-plan
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

## Testing Strategies

### Security Testing

Implement security testing with GitLab's security scanners:

```yaml
include:
  - template: SAST.gitlab-ci.yml
  - template: Dependency-Scanning.gitlab-ci.yml
  - template: Container-Scanning.gitlab-ci.yml
  - template: DAST.gitlab-ci.yml

variables:
  DAST_WEBSITE: https://example.com
```

## Best Practices

### Pipeline Structure

- Split large pipelines into parent/child pipelines
- Use YAML anchors and templates for reusable configurations
- Keep job scripts short; move complex logic to separate scripts
- Use descriptive job and stage names

### Performance

- Use directed acyclic graphs (`needs`) instead of just stages
- Mark jobs as interruptible when possible
- Use job dependencies wisely to minimize artifact transfers
- Use efficient cache strategies

### Reliability

- Implement retry strategies for flaky tests
- Use idempotent scripts for deployments
- Include proper error handling in scripts
- Use timeouts to prevent jobs from hanging

## Next Steps

Now that you understand advanced CI/CD techniques in GitLab, let's look at [Real-World Examples](10-examples.md) to see how to apply these concepts in practice.
