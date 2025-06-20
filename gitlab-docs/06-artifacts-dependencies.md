# Artifacts and Dependencies

Artifacts and dependencies are crucial components of GitLab CI/CD that allow you to share data between jobs and preserve build outputs. This document explains how to effectively use them in your pipelines.

## Artifacts

Artifacts are files or directories that are generated during a job and then saved for later use. They can be:

- Downloaded manually from the GitLab UI
- Passed to subsequent jobs in the pipeline
- Automatically deployed to environments
- Stored as release assets

### Basic Artifact Configuration

To create artifacts from a job, specify the paths you want to save in your `.gitlab-ci.yml` file:

```yaml
build-job:
  stage: build
  script:
    - echo "Building the application..."
    - mkdir -p build/
    - echo "Built application" > build/app.txt
  artifacts:
    paths:
      - build/
```

### Artifact Expiration

By default, artifacts are stored for 30 days. You can customize this with the `expire_in` parameter:

```yaml
build-job:
  script:
    - echo "Building the application..."
  artifacts:
    paths:
      - build/
    expire_in: 1 week
```

Valid time units include:
- `seconds`
- `minutes`
- `hours`
- `days`
- `weeks`
- `months`
- `years`

You can also set artifacts to never expire with `expire_in: never`.

### Artifact Name

You can provide a custom name for your artifacts:

```yaml
build-job:
  script:
    - echo "Building the application..."
  artifacts:
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_NAME"
    paths:
      - build/
```

This creates an artifact named like `build-job-main.zip`.

### When to Create Artifacts

By default, artifacts are created only when a job succeeds. You can change this behavior with the `when` parameter:

```yaml
test-job:
  script:
    - echo "Running tests..."
    - mkdir -p reports/
    - echo "Test results" > reports/results.txt
  artifacts:
    paths:
      - reports/
    when: always  # Create artifacts even if the job fails
```

Options for the `when` parameter:
- `on_success` (default): Only create artifacts when the job succeeds
- `on_failure`: Only create artifacts when the job fails
- `always`: Always create artifacts, regardless of job status

### Artifact Types

GitLab supports different types of artifacts that integrate with its UI:

#### Reports

Report artifacts display information in the GitLab UI:

```yaml
test-job:
  script:
    - echo "Running tests..."
    - mkdir -p reports/
    - echo "<testsuite><testcase>...</testcase></testsuite>" > reports/junit.xml
  artifacts:
    reports:
      junit: reports/junit.xml
```

Common report types:
- `junit`: Test results (displays in the Test tab)
- `coverage_report`: Code coverage results
- `codequality`: Code quality issues
- `sast`: Security scanning results
- `dast`: Dynamic security testing results
- `performance`: Performance testing data

#### Archive vs. Raw Artifacts

By default, GitLab compresses artifacts into a zip archive. To prevent this for binary files, use the `untracked` option:

```yaml
build-job:
  script:
    - echo "Building the application..."
  artifacts:
    untracked: true  # Include all files not tracked by Git
```

### Artifact Size Limits

GitLab has limits on artifact sizes:
- GitLab.com: 1GB per artifact by default
- Self-managed: Configurable by administrators

## Dependencies

Dependencies allow you to specify which jobs a particular job depends on. This controls:

1. Job execution order
2. Access to artifacts from previous jobs

### Basic Dependencies

To explicitly define dependencies:

```yaml
stages:
  - build
  - test

build-job:
  stage: build
  script:
    - echo "Building the application..."
    - mkdir -p build/
    - echo "Built application" > build/app.txt
  artifacts:
    paths:
      - build/

test-job:
  stage: test
  script:
    - echo "Testing the application..."
    - cat build/app.txt
  dependencies:
    - build-job  # This job depends on build-job and will have access to its artifacts
```

### Disabling Artifact Passing

You can prevent a job from downloading artifacts from its dependencies:

```yaml
test-job:
  stage: test
  script:
    - echo "Testing without artifacts..."
  dependencies: []  # Empty array means no dependencies
```

### Specifying Dependencies for Specific Jobs

When a job has multiple dependencies, you can specify which ones to get artifacts from:

```yaml
stages:
  - build
  - test
  - deploy

build-backend:
  stage: build
  script:
    - echo "Building backend..."
    - mkdir -p backend/
    - echo "Backend build" > backend/app.txt
  artifacts:
    paths:
      - backend/

build-frontend:
  stage: build
  script:
    - echo "Building frontend..."
    - mkdir -p frontend/
    - echo "Frontend build" > frontend/app.txt
  artifacts:
    paths:
      - frontend/

test-backend:
  stage: test
  script:
    - echo "Testing backend..."
    - cat backend/app.txt
  dependencies:
    - build-backend  # Only download backend artifacts

deploy-job:
  stage: deploy
  script:
    - echo "Deploying application..."
    - cat backend/app.txt
    - cat frontend/app.txt
  dependencies:
    - build-backend
    - build-frontend  # Download artifacts from both build jobs
```

## Caching vs. Artifacts

Caching and artifacts serve different purposes:

### Artifacts

- Passed between jobs and stages
- Downloadable from GitLab UI
- Specific to a pipeline and job
- Used for build outputs, reports, etc.

### Cache

- Shared between pipeline runs
- Not downloadable from GitLab UI
- Used for dependencies, packages, etc.
- Optimizes build time by reusing previous downloads

Example using both:

```yaml
build-job:
  script:
    - echo "Building the application..."
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/  # Build output to share between jobs
  cache:
    paths:
      - node_modules/  # Cache node modules between pipeline runs
```

## Accessing Artifacts

### Downloading Artifacts Manually

You can download artifacts:
1. From the job page
2. From the pipeline page
3. Via the GitLab API

### Using Artifacts in Jobs

When a job depends on another job, artifacts are automatically downloaded to the job's working directory:

```yaml
test-job:
  script:
    - echo "Testing the application..."
    - ls -la  # Artifacts from dependencies are in the job's working directory
```

### Artifacts API

You can access artifacts programmatically via the GitLab API:

```bash
# Download artifact from the latest successful pipeline on the main branch
curl --header "PRIVATE-TOKEN: <your_token>" \
     "https://gitlab.com/api/v4/projects/<project_id>/jobs/artifacts/main/download?job=build-job"
```

## Advanced Artifact Usage

### Conditional Artifacts

Create artifacts only under specific conditions:

```yaml
build-job:
  script:
    - echo "Building the application..."
  artifacts:
    paths:
      - build/
    when: on_success
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
      artifacts:
        expire_in: 1 month  # Override expiration for main branch
    - when: always
```

### Excluding Files from Artifacts

You can exclude specific files from artifacts:

```yaml
build-job:
  script:
    - echo "Building the application..."
    - mkdir -p build/logs/
    - echo "Debug info" > build/logs/debug.log
    - echo "App binary" > build/app.bin
  artifacts:
    paths:
      - build/
    exclude:
      - build/logs/  # Don't include logs in artifacts
```

### Release Assets from Artifacts

You can attach job artifacts to GitLab releases:

```yaml
build-release:
  script:
    - echo "Building release..."
    - echo "v1.0.0" > VERSION
  artifacts:
    name: "release-${CI_COMMIT_TAG}"
    paths:
      - VERSION
  release:
    tag_name: $CI_COMMIT_TAG
    description: "Release $CI_COMMIT_TAG"
    assets:
      links:
        - name: "Version file"
          url: "${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/raw/VERSION"
```

## Practical Examples

### Example: Build and Deploy a Web Application

```yaml
stages:
  - build
  - test
  - deploy

build-frontend:
  stage: build
  script:
    - echo "Installing dependencies..."
    - echo "Building frontend..."
    - mkdir -p dist/
    - echo "<html>App</html>" > dist/index.html
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

test-frontend:
  stage: test
  script:
    - echo "Testing frontend..."
    - grep "App" dist/index.html
  dependencies:
    - build-frontend

deploy-staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
    - echo "Deploying files:"
    - ls -la dist/
  dependencies:
    - build-frontend
  environment:
    name: staging
    url: https://staging.example.com
```

### Example: Generate and Publish Documentation

```yaml
stages:
  - build
  - deploy

build-docs:
  stage: build
  script:
    - echo "Generating documentation..."
    - mkdir -p docs/
    - echo "# Documentation" > docs/index.md
    - echo "## API Reference" > docs/api.md
  artifacts:
    paths:
      - docs/

pages:
  stage: deploy
  script:
    - echo "Publishing documentation..."
    - mkdir -p public/
    - cp -r docs/* public/
  artifacts:
    paths:
      - public/
  dependencies:
    - build-docs
  only:
    - main
```

## Best Practices

### Optimize Artifact Size

- Only include necessary files
- Use `.gitignore`-style patterns to exclude irrelevant files
- Consider using multiple jobs with specific artifacts instead of one job with everything

### Use Expiration Wisely

- Set shorter expiration for frequently changing branches
- Set longer expiration (or never) for important artifacts like releases
- Consider your storage costs and limits

### Manage Dependencies Properly

- Only specify dependencies you actually need
- Use empty dependencies (`dependencies: []`) when artifacts aren't needed

### Consider Your CI/CD Minutes and Bandwidth

- Large artifacts can slow down your pipelines
- Use caching for dependencies instead of artifacts
- Use sensible expiration times to manage storage

## Next Steps

Now that you understand artifacts and dependencies, let's explore [Environments and Deployments](07-environments-deployments.md) in more detail. 