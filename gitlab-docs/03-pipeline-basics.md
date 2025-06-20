# Pipeline Configuration Basics

GitLab CI/CD pipelines are configured using a YAML file called `.gitlab-ci.yml` placed in the root of your repository. This file defines the structure and execution of your CI/CD pipeline.

## Pipeline Structure

A basic pipeline consists of:

- **Jobs**: Individual units of work that perform specific tasks
- **Stages**: Groups of jobs that run in sequence
- **Scripts**: Commands that execute within jobs
- **Rules**: Conditions that determine when jobs run

## The `.gitlab-ci.yml` File

Here's an anatomy of a simple `.gitlab-ci.yml` file:

```yaml
# Define stages and their execution order
stages:
  - build
  - test
  - deploy

# Define variables available to all jobs
variables:
  PROJECT_NAME: "my-awesome-project"

# A job in the build stage
build-job:
  stage: build
  script:
    - echo "Building $PROJECT_NAME..."
    - mkdir -p build
    - touch build/info.txt
  artifacts:
    paths:
      - build/

# A job in the test stage
test-job:
  stage: test
  script:
    - echo "Testing $PROJECT_NAME..."
    - test -f build/info.txt
  dependencies:
    - build-job

# A job in the deploy stage
deploy-job:
  stage: deploy
  script:
    - echo "Deploying $PROJECT_NAME..."
    - echo "Deployment successful!"
  environment:
    name: production
```

## Pipeline Execution

When you commit and push this file to your repository, GitLab automatically detects it and triggers a pipeline. The pipeline runs through each stage in order, executing all jobs within a stage in parallel (if there are multiple runners available).

## Default Pipeline Stages

GitLab provides default stages that you can use or override:

```
.pre → build → test → deploy → .post
```

The `.pre` and `.post` stages are special stages that run automatically before and after the defined stages.

## Types of Pipelines

GitLab supports different types of pipelines:

### Branch Pipelines

- Triggered when you push changes to a branch
- Run the entire pipeline configuration

### Merge Request Pipelines

- Triggered when you create or update a merge request
- Can be configured to run specific jobs relevant to code review

### Scheduled Pipelines

- Run at specified times (like cron jobs)
- Useful for nightly builds, scheduled deployments, or periodic tasks

### Manual Pipelines

- Triggered manually by a user
- Useful for deploying to production or other critical environments

## Pipeline Trigger Events

Pipelines can be triggered by various events:

- Code pushes
- Merge request creation or updates
- Tags creation
- Scheduled runs
- API calls
- Web interface triggers
- Pipeline triggers (using tokens)

## Example: Configuring Different Trigger Events

```yaml
build-job:
  stage: build
  script:
    - echo "Building the app..."
  # Run this job only when changes are pushed to the main branch
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always

test-job:
  stage: test
  script:
    - echo "Running tests..."
  # Run this job for all branches and merge requests
  rules:
    - when: always

deploy-staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
  # Run this job only on the main branch
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
  environment:
    name: staging

deploy-production:
  stage: deploy
  script:
    - echo "Deploying to production..."
  # Run this job only when a tag is created and make it manual
  rules:
    - if: '$CI_COMMIT_TAG'
      when: manual
  environment:
    name: production
```

## Job Control Using Rules

The `rules` keyword allows you to define conditions for when a job should run. This powerful feature enables you to create complex workflows based on branch names, commit messages, changes to specific files, and more.

### Common Rule Conditions

- `if: '$CI_COMMIT_BRANCH == "main"'` - Run on main branch
- `if: '$CI_PIPELINE_SOURCE == "merge_request_event"'` - Run on merge requests
- `if: '$CI_COMMIT_TAG'` - Run when a tag is created
- `changes: ['*.js', '*.jsx']` - Run when JavaScript files change

### Rule Actions

- `when: always` - Always run the job
- `when: never` - Never run the job
- `when: manual` - Run the job only when manually triggered
- `when: on_success` - Run the job only when previous stages succeed
- `when: on_failure` - Run the job only when a previous job fails

## Workflow Control

The `workflow` keyword at the top level allows you to define conditions for when a pipeline should run:

```yaml
workflow:
  rules:
    - if: '$CI_COMMIT_TAG'
      when: always
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - when: never  # Skip pipelines for all other cases
```

## GitLab CI/CD Predefined Variables

GitLab provides many predefined variables that you can use in your pipeline configuration. These include information about the repository, commit, branch, merge request, and more.

Common predefined variables include:

- `CI_COMMIT_BRANCH`: The branch name
- `CI_COMMIT_SHA`: The commit SHA
- `CI_COMMIT_TAG`: The tag name (if any)
- `CI_PIPELINE_SOURCE`: The source of the pipeline trigger
- `CI_PROJECT_DIR`: The directory where the repository is cloned

You can find a complete list of predefined variables in the [GitLab documentation](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html).

## Next Steps

Now that you understand the basics of pipeline configuration, let's explore [Jobs and Stages](04-jobs-stages.md) in more detail. 