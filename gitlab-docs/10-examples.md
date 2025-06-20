# Real-World Examples

This document provides complete, practical examples of GitLab CI/CD pipelines for different types of projects. These examples demonstrate how to apply the concepts covered in previous sections to real-world scenarios.

## Node.js Web Application

This example shows a complete CI/CD pipeline for a Node.js web application:

```yaml
stages:
  - install
  - test
  - build
  - deploy

variables:
  NODE_ENV: "development"
  CACHE_KEY: "$CI_COMMIT_REF_SLUG"

# Cache node_modules across jobs
cache:
  key: "$CACHE_KEY"
  paths:
    - node_modules/

install-dependencies:
  stage: install
  image: node:16-alpine
  script:
    - npm ci
  artifacts:
    paths:
      - node_modules/

lint:
  stage: test
  image: node:16-alpine
  script:
    - npm run lint
  dependencies:
    - install-dependencies

unit-tests:
  stage: test
  image: node:16-alpine
  script:
    - npm run test:unit
  dependencies:
    - install-dependencies
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      junit: junit-test-results.xml

build-app:
  stage: build
  image: node:16-alpine
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
  dependencies:
    - install-dependencies
  only:
    - main
    - tags

deploy-staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client rsync
    - eval $(ssh-agent -s)
    - echo "$STAGING_SSH_PRIVATE_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - ssh-keyscan $STAGING_SERVER >> ~/.ssh/known_hosts
  script:
    - rsync -avz --delete dist/ $STAGING_USER@$STAGING_SERVER:$STAGING_PATH
  environment:
    name: staging
    url: https://staging.example.com
  dependencies:
    - build-app
  only:
    - main

deploy-production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client rsync
    - eval $(ssh-agent -s)
    - echo "$PRODUCTION_SSH_PRIVATE_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - ssh-keyscan $PRODUCTION_SERVER >> ~/.ssh/known_hosts
  script:
    - rsync -avz --delete dist/ $PRODUCTION_USER@$PRODUCTION_SERVER:$PRODUCTION_PATH
  environment:
    name: production
    url: https://example.com
  dependencies:
    - build-app
  only:
    - tags
  when: manual
```

This pipeline:
1. Installs dependencies
2. Runs linting and tests in parallel
3. Builds the application for production
4. Deploys to staging automatically for the main branch
5. Deploys to production manually for tags

## Python Django Application with Docker

This example shows a CI/CD pipeline for a Python Django application using Docker:

```yaml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_REGISTRY: registry.example.com
  IMAGE_NAME: $DOCKER_REGISTRY/my-group/django-app
  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""

test:
  stage: test
  image: python:3.9
  services:
    - postgres:13
  variables:
    POSTGRES_DB: test_db
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: postgres
    DATABASE_URL: "postgres://postgres:postgres@postgres:5432/test_db"
  before_script:
    - pip install -r requirements.txt
    - pip install pytest pytest-django pytest-cov
  script:
    - python manage.py migrate
    - pytest --cov=app
  coverage: '/TOTAL.+?(\d+\%)/'

lint:
  stage: test
  image: python:3.9
  before_script:
    - pip install flake8 black isort
  script:
    - flake8 .
    - black --check .
    - isort --check-only --profile black .

build-image:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
    - |
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        docker tag $IMAGE_NAME:$CI_COMMIT_SHA $IMAGE_NAME:latest
        docker push $IMAGE_NAME:latest
      fi
  only:
    - main
    - tags
    - merge_requests

deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default
    - sed -i "s|__IMAGE__|$IMAGE_NAME:$CI_COMMIT_SHA|g" kubernetes/staging/deployment.yaml
    - kubectl apply -f kubernetes/staging/
    - kubectl rollout status deployment/django-app -n staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default
    - sed -i "s|__IMAGE__|$IMAGE_NAME:$CI_COMMIT_TAG|g" kubernetes/production/deployment.yaml
    - kubectl apply -f kubernetes/production/
    - kubectl rollout status deployment/django-app -n production
  environment:
    name: production
    url: https://example.com
  only:
    - tags
  when: manual
```

This pipeline:
1. Runs tests and linting in parallel
2. Builds a Docker image and tags it with the commit SHA
3. Deploys to staging automatically for the main branch
4. Deploys to production manually for tags

## Microservices with Kubernetes

This example demonstrates a parent pipeline that triggers child pipelines for multiple microservices:

```yaml
# Parent .gitlab-ci.yml
stages:
  - generate
  - build
  - deploy

variables:
  SERVICES: "api frontend auth worker"

generate-pipelines:
  stage: generate
  image: alpine:latest
  script:
    - |
      mkdir -p .generated
      for service in $SERVICES; do
        if [ -d "services/$service" ] && [ -f "services/$service/.gitlab-ci.yml" ]; then
          echo "Preparing pipeline for $service"
          cp "services/$service/.gitlab-ci.yml" ".generated/$service.yml"
          # Add necessary variables to each child pipeline
          echo "variables:" >> ".generated/$service.yml"
          echo "  SERVICE_NAME: \"$service\"" >> ".generated/$service.yml"
        fi
      done
  artifacts:
    paths:
      - .generated/

trigger-service-pipelines:
  stage: build
  trigger:
    strategy: depend
    include:
      - artifact: .generated/api.yml
        job: generate-pipelines
      - artifact: .generated/frontend.yml
        job: generate-pipelines
      - artifact: .generated/auth.yml
        job: generate-pipelines
      - artifact: .generated/worker.yml
        job: generate-pipelines

deploy-all:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default
    - kubectl apply -f kubernetes/common/
  environment:
    name: production
  only:
    - tags
  when: manual
```

Child pipeline for a service (e.g., `services/api/.gitlab-ci.yml`):

```yaml
# Child pipeline for API service
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_REGISTRY: registry.example.com
  IMAGE_NAME: $DOCKER_REGISTRY/my-group/$SERVICE_NAME

test:
  stage: test
  image: node:16-alpine
  script:
    - cd services/$SERVICE_NAME
    - npm ci
    - npm test

build-image:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  script:
    - cd services/$SERVICE_NAME
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $DOCKER_REGISTRY
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
    - |
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        docker tag $IMAGE_NAME:$CI_COMMIT_SHA $IMAGE_NAME:latest
        docker push $IMAGE_NAME:latest
      fi

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials gitlab --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=gitlab
    - kubectl config use-context default
    - sed -i "s|__IMAGE__|$IMAGE_NAME:$CI_COMMIT_SHA|g" services/$SERVICE_NAME/kubernetes/deployment.yaml
    - kubectl apply -f services/$SERVICE_NAME/kubernetes/
    - kubectl rollout status deployment/$SERVICE_NAME
  environment:
    name: production/$SERVICE_NAME
  only:
    - tags
  when: manual
```

This pipeline structure:
1. Parent pipeline generates child pipelines for each service
2. Each service has its own dedicated pipeline with test, build, and deploy stages
3. A common deployment job can deploy shared resources

## Infrastructure as Code with Terraform

This example shows a pipeline for infrastructure management with Terraform:

```yaml
stages:
  - validate
  - plan
  - apply
  - destroy

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/terraform
  TF_STATE_NAME: default

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - ${TF_ROOT}/.terraform

before_script:
  - cd ${TF_ROOT}
  - terraform init -backend-config="key=${TF_STATE_NAME}"

validate:
  stage: validate
  image: hashicorp/terraform:latest
  script:
    - terraform validate
    - terraform fmt -check

plan:
  stage: plan
  image: hashicorp/terraform:latest
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week

apply:
  stage: apply
  image: hashicorp/terraform:latest
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
    - if: '$CI_COMMIT_TAG'
      when: manual

destroy:
  stage: destroy
  image: hashicorp/terraform:latest
  script:
    - terraform destroy -auto-approve
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
  environment:
    name: infrastructure
    action: stop
```

This pipeline:
1. Validates Terraform configuration
2. Generates an execution plan
3. Applies infrastructure changes manually
4. Allows for infrastructure destruction as needed

## Best Practices from These Examples

From these real-world examples, here are some best practices to follow:

1. **Use stages effectively**: Structure your pipeline into logical stages that represent your workflow.

2. **Cache dependencies**: Cache dependencies to speed up subsequent pipeline runs.

3. **Use appropriate Docker images**: Choose lightweight, specific images for each job.

4. **Separate environments**: Clearly separate staging and production environments.

5. **Manual deployment gates**: Use manual triggers for critical deployments.

6. **Artifacts management**: Use artifacts to pass data between jobs and stages.

7. **Secure handling of credentials**: Use masked and protected variables for sensitive information.

8. **Testing at multiple levels**: Include unit tests, integration tests, and quality checks.

9. **Environment-specific configurations**: Adjust application configurations for different environments.

10. **CI/CD security**: Include security scanning in your pipeline.

11. **Service-specific pipelines**: Use parent/child pipelines for microservices.

12. **Infrastructure as Code**: Include infrastructure management in your pipeline.

These examples demonstrate how GitLab CI/CD can be applied to various project types, from web applications to mobile apps, and from monolithic applications to microservices.
