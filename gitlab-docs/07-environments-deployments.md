# Environments and Deployments

GitLab Environments provide a way to track your deployments and monitor their status across different infrastructure targets. This document explains how to configure and use environments for effective CI/CD deployments.

## What are GitLab Environments?

An environment in GitLab represents a deployment target, such as production, staging, or development. Environments allow you to:

- Track deployments to different targets
- View deployment history
- Monitor environment status
- Implement controlled deployment workflows
- Automate infrastructure management

## Basic Environment Configuration

To define an environment in your `.gitlab-ci.yml`, add the `environment` keyword to your job:

```yaml
deploy-staging:
  stage: deploy
  script:
    - echo "Deploying to staging server..."
  environment:
    name: staging
```

This simple configuration creates a "staging" environment in GitLab and tracks each deployment.

## Environment URL

You can associate a URL with your environment to make it easily accessible from the GitLab interface:

```yaml
deploy-production:
  stage: deploy
  script:
    - echo "Deploying to production server..."
  environment:
    name: production
    url: https://example.com
```

This adds a direct link to your deployed application in the GitLab environment page.

## Environment Tiers

GitLab recognizes certain environment names as specific tiers:

- `development`: For development environments
- `testing`: For testing environments
- `staging`: For staging environments
- `production`: For production environments

Using these predefined names helps GitLab organize your environments in the UI.

## Environment Deployment

### Basic Deployment

A basic deployment job that uses an environment looks like this:

```yaml
deploy-job:
  stage: deploy
  script:
    - echo "Deploying application..."
    - rsync -avz build/ user@server:/var/www/html/
  environment:
    name: production
    url: https://example.com
```

### Deployment Safety

To make deployments safer, you can use:

#### Manual Deployments

```yaml
deploy-production:
  stage: deploy
  script:
    - echo "Deploying to production..."
  environment:
    name: production
    url: https://example.com
  rules:
    - when: manual  # Requires manual action to deploy
```

#### Deployment Approvals

For more critical environments, you can require approvals before deployment:

```yaml
deploy-production:
  stage: deploy
  script:
    - echo "Deploying to production..."
  environment:
    name: production
    url: https://example.com
  rules:
    - when: manual
  needs:
    - job: test-job
      artifacts: true
```

When combined with [approval rules](https://docs.gitlab.com/ee/ci/environments/deployment_approvals.html) in the project settings, this creates a safe deployment process with required reviews.

## Dynamic Environments

You can create dynamic environments based on branch names or other variables:

```yaml
deploy-review:
  stage: deploy
  script:
    - echo "Deploying review app for branch $CI_COMMIT_REF_NAME..."
    - ./deploy-script review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: https://$CI_COMMIT_REF_SLUG.example.com
    on_stop: stop-review
    auto_stop_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH != "main"
```

This creates a unique environment for each branch, making it easy to preview changes.

### Stopping Dynamic Environments

You can define a job to stop and clean up dynamic environments:

```yaml
stop-review:
  stage: deploy
  script:
    - echo "Stopping review app for branch $CI_COMMIT_REF_NAME..."
    - ./cleanup-script review-$CI_COMMIT_REF_SLUG
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  rules:
    - if: $CI_COMMIT_BRANCH != "main"
      when: manual
```

## Environment-specific Variables

You can define variables specific to each environment:

```yaml
deploy-staging:
  stage: deploy
  script:
    - echo "Deploying to $CI_ENVIRONMENT_NAME with URL $CI_ENVIRONMENT_URL"
    - echo "Using $DB_HOST as database host"
  environment:
    name: staging
    url: https://staging.example.com
  variables:
    DB_HOST: staging-db.example.com
```

In GitLab, you can also define environment-scoped CI/CD variables in the project settings.

## Deployment History and Rollbacks

GitLab keeps track of all deployments to each environment, allowing you to:

- See when deployments occurred
- Identify who triggered each deployment
- View the commit and branch that was deployed
- Roll back to a previous deployment

To enable rollbacks, make your deployment script compatible with re-deploying older versions:

```yaml
deploy-production:
  stage: deploy
  script:
    - echo "Deploying $CI_COMMIT_SHA to production..."
    - git checkout $CI_COMMIT_SHA
    - ./deploy-script
  environment:
    name: production
    url: https://example.com
  rules:
    - when: manual
```

## Deployment Strategies

### Basic Deployment

The simplest deployment strategy is to directly update your application:

```yaml
deploy-job:
  script:
    - echo "Deploying application..."
    - scp -r build/* user@server:/var/www/html/
  environment:
    name: production
```

### Blue-Green Deployment

Blue-green deployment involves maintaining two identical production environments:

```yaml
deploy-production:
  script:
    - echo "Preparing new environment..."
    - ./setup-new-environment
    - echo "Deploying to new environment..."
    - ./deploy-to-new-environment
    - echo "Running tests on new environment..."
    - ./test-new-environment
    - echo "Switching traffic to new environment..."
    - ./switch-traffic
  environment:
    name: production
    url: https://example.com
```

### Canary Deployment

Canary deployment gradually routes traffic to the new version:

```yaml
deploy-canary:
  script:
    - echo "Deploying canary version..."
    - ./deploy-canary 10%  # Deploy to 10% of users
  environment:
    name: production/canary
    url: https://example.com
    on_stop: stop-canary

deploy-production:
  script:
    - echo "Deploying to 100% of users..."
    - ./deploy-full
  environment:
    name: production
    url: https://example.com
  needs:
    - job: deploy-canary
  when: manual
```

## Environment Monitoring

GitLab can integrate with monitoring systems to display metrics for your environments:

```yaml
deploy-production:
  script:
    - echo "Deploying to production..."
  environment:
    name: production
    url: https://example.com
    metrics_path: /prometheus/metrics  # Path for metrics
```

## Kubernetes Integration

GitLab has deep integration with Kubernetes, allowing for seamless deployments:

```yaml
deploy-k8s:
  stage: deploy
  script:
    - echo "Deploying to Kubernetes..."
    - kubectl apply -f kubernetes/deployment.yaml
  environment:
    name: production
    url: https://example.com
    kubernetes:
      namespace: production
```

With the GitLab Kubernetes Agent, you can deploy to multiple clusters and manage complex deployments.

## Environment Grouping and Advanced Features

### Parent-Child Environments

You can create hierarchical environments by using a slash in the environment name:

```yaml
deploy-region-1:
  script:
    - echo "Deploying to US data center..."
  environment:
    name: production/us
    url: https://us.example.com

deploy-region-2:
  script:
    - echo "Deploying to EU data center..."
  environment:
    name: production/eu
    url: https://eu.example.com
```

This creates "us" and "eu" as child environments of "production".

## Practical Examples

### Web Application Deployment Pipeline

This example shows a complete pipeline for deploying a web application to multiple environments:

```yaml
stages:
  - build
  - test
  - deploy

variables:
  APP_NAME: "my-awesome-app"

build-app:
  stage: build
  script:
    - echo "Building $APP_NAME..."
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/

test-app:
  stage: test
  script:
    - echo "Testing $APP_NAME..."
    - npm test
  dependencies:
    - build-app

deploy-staging:
  stage: deploy
  script:
    - echo "Deploying $APP_NAME to staging..."
    - rsync -avz dist/ deploy@staging-server:/var/www/$APP_NAME/
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: on_success

deploy-production:
  stage: deploy
  script:
    - echo "Deploying $APP_NAME to production..."
    - rsync -avz dist/ deploy@production-server:/var/www/$APP_NAME/
  environment:
    name: production
    url: https://example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
  needs:
    - deploy-staging
```

### Microservices Deployment

This example shows deploying multiple microservices to a Kubernetes cluster:

```yaml
stages:
  - build
  - test
  - deploy

.deploy-template: &deploy-template
  stage: deploy
  script:
    - echo "Deploying ${SERVICE_NAME} to ${CI_ENVIRONMENT_NAME}..."
    - kubectl -n ${KUBE_NAMESPACE} set image deployment/${SERVICE_NAME} ${SERVICE_NAME}=${SERVICE_IMAGE}:${CI_COMMIT_SHA}
  dependencies: []

# Build and deploy API service
build-api:
  stage: build
  script:
    - echo "Building API service..."
    - docker build -t ${CI_REGISTRY_IMAGE}/api:${CI_COMMIT_SHA} ./api
    - docker push ${CI_REGISTRY_IMAGE}/api:${CI_COMMIT_SHA}

deploy-api-staging:
  <<: *deploy-template
  variables:
    SERVICE_NAME: api
    SERVICE_IMAGE: ${CI_REGISTRY_IMAGE}/api
    KUBE_NAMESPACE: staging
  environment:
    name: staging/api
    url: https://api.staging.example.com

deploy-api-production:
  <<: *deploy-template
  variables:
    SERVICE_NAME: api
    SERVICE_IMAGE: ${CI_REGISTRY_IMAGE}/api
    KUBE_NAMESPACE: production
  environment:
    name: production/api
    url: https://api.example.com
  rules:
    - when: manual

# Build and deploy Web service
build-web:
  stage: build
  script:
    - echo "Building Web service..."
    - docker build -t ${CI_REGISTRY_IMAGE}/web:${CI_COMMIT_SHA} ./web
    - docker push ${CI_REGISTRY_IMAGE}/web:${CI_COMMIT_SHA}

deploy-web-staging:
  <<: *deploy-template
  variables:
    SERVICE_NAME: web
    SERVICE_IMAGE: ${CI_REGISTRY_IMAGE}/web
    KUBE_NAMESPACE: staging
  environment:
    name: staging/web
    url: https://staging.example.com

deploy-web-production:
  <<: *deploy-template
  variables:
    SERVICE_NAME: web
    SERVICE_IMAGE: ${CI_REGISTRY_IMAGE}/web
    KUBE_NAMESPACE: production
  environment:
    name: production/web
    url: https://example.com
  rules:
    - when: manual
```

## Best Practices

### Environment Naming

- Use consistent naming patterns
- Follow the tier structure (development, testing, staging, production)
- Use parent-child relationships for related environments

### Deployment Safety

- Use manual deployments for critical environments
- Implement approval requirements for production deployments
- Add automated tests before allowing deployments

### Environment Management

- Clean up temporary environments
- Use `auto_stop_in` for review environments
- Document environment URLs and access methods
- Set up proper monitoring for all environments

### Security

- Restrict environment access with protection rules
- Use secure methods to pass deployment credentials
- Never expose sensitive data in deployment logs

## Next Steps

Now that you understand environments and deployments, let's explore [Variables and Secrets](08-variables-secrets.md) to learn how to securely manage sensitive information in your CI/CD pipelines. 