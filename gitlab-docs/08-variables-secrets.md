# Variables and Secrets

Properly managing variables and secrets is crucial for secure and effective CI/CD pipelines. This document explains how to configure, use, and protect sensitive information in GitLab CI/CD.

## Types of CI/CD Variables

GitLab offers several types of variables for use in your pipelines:

### Predefined Variables

GitLab automatically provides many [predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) that contain information about the project, pipeline, job, and environment:

```yaml
job-info:
  script:
    - echo "Project: $CI_PROJECT_NAME"
    - echo "Branch: $CI_COMMIT_BRANCH"
    - echo "Commit: $CI_COMMIT_SHORT_SHA"
    - echo "Pipeline ID: $CI_PIPELINE_ID"
```

Common predefined variables include:
- `CI_PROJECT_PATH`: Full project path (namespace/project)
- `CI_COMMIT_SHA`: Full commit SHA
- `CI_COMMIT_SHORT_SHA`: First 8 characters of commit SHA
- `CI_COMMIT_BRANCH`: Branch name (only in branch pipelines)
- `CI_COMMIT_TAG`: Tag name (only in tag pipelines)
- `CI_PIPELINE_ID`: Pipeline ID
- `CI_JOB_ID`: Job ID
- `CI_ENVIRONMENT_NAME`: Name of the environment (if specified)

### Custom Variables in .gitlab-ci.yml

You can define your own variables directly in the `.gitlab-ci.yml` file:

```yaml
variables:
  APP_NAME: "my-awesome-app"
  APP_VERSION: "1.0.0"

build-job:
  variables:
    BUILD_TYPE: "release"
  script:
    - echo "Building $APP_NAME version $APP_VERSION ($BUILD_TYPE)"
```

Variables defined at the top level are available to all jobs, while variables defined in a job are scoped to that job only.

### Project and Group Variables

You can also define variables in the GitLab UI at the project or group level. These are ideal for:
- Sensitive information (credentials, API keys)
- Environment-specific configuration
- Values that change frequently

## Setting Up CI/CD Variables

### In the GitLab UI

To add variables through the GitLab UI:

1. Navigate to your project
2. Go to **Settings > CI/CD**
3. Expand the **Variables** section
4. Click **Add Variable**
5. Enter the key, value, and other options
6. Click **Add Variable**

### Variable Options

When creating variables in the GitLab UI, you have several options:

- **Type**: Variable or File (for multi-line content)
- **Protect variable**: Only available for protected branches and tags
- **Mask variable**: Hide the value in job logs
- **Expand variable**: Whether to expand CI/CD variable references in the value
- **Scope environments**: Limit the variable to specific environments

## Variable Security Features

### Protected Variables

Protected variables are only available in jobs running on protected branches or tags:

```yaml
deploy-production:
  script:
    - echo "Deploying with protected API key: $PROTECTED_API_KEY"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

This helps prevent access to sensitive credentials from untrusted code.

### Masked Variables

Masked variables are hidden in job logs (replaced with `[MASKED]`):

```
# A job log with a masked variable
$ echo "Using API key: [MASKED]"
```

To be maskable, the variable value must:
- Be a single line string
- Be at most 255 characters long
- Not consist solely of whitespace
- Not be one of the predefined variables
- Not match an environment variable value used by the Runner

### File Variables

For multi-line content or larger secrets, use file variables:

```yaml
deploy-job:
  script:
    - echo "Using SSH key from file:"
    - cat $SSH_KEY_FILE
    - scp -i $SSH_KEY_FILE app.tar.gz user@server:/var/www/
```

The variable content is stored in a temporary file, and the variable name contains the path to this file.

## Environment-Scoped Variables

You can scope variables to specific environments:

```yaml
deploy-staging:
  script:
    - echo "Deploying to $CI_ENVIRONMENT_NAME with URL $DEPLOY_URL"
    # This uses the staging DEPLOY_URL value
  environment:
    name: staging

deploy-production:
  script:
    - echo "Deploying to $CI_ENVIRONMENT_NAME with URL $DEPLOY_URL"
    # This uses the production DEPLOY_URL value
  environment:
    name: production
```

In the GitLab UI, you can scope variables to specific environments by selecting the environment in the environment scope dropdown when creating or editing a variable.

## Variable Precedence

When variables with the same name are defined in multiple places, GitLab uses this precedence order (highest to lowest):

1. Trigger variables and scheduled pipeline variables
2. Project-level variables
3. Group-level variables
4. YAML-defined job-level variables
5. YAML-defined global variables
6. Predefined variables
7. Environment variables

## Using Variables in Scripts

### Basic Usage

Variables are used in scripts just like regular shell variables:

```yaml
build-job:
  variables:
    APP_VERSION: "1.0.0"
  script:
    - echo "Building version $APP_VERSION"
    - sed -i "s/VERSION/$APP_VERSION/g" config.json
```

### Advanced Variable Expansion

GitLab supports variable interpolation within other variables:

```yaml
variables:
  PROJECT: "my-app"
  ENVIRONMENT: "production"
  CONFIG_PATH: "${PROJECT}-${ENVIRONMENT}.yml"

job:
  script:
    - echo "Loading config from $CONFIG_PATH"  # Outputs: Loading config from my-app-production.yml
```

## Managing Secrets

### Storing Secrets

Best practices for storing secrets:

1. **Always use masked variables** for sensitive values
2. **Use protected variables** for production credentials
3. **Scope variables to environments** when applicable
4. **Use file variables** for certificates, keys, and long texts
5. **Consider using an external secrets manager** for very sensitive values

### External Secrets Management

For enhanced security, consider integrating with an external secrets manager:

```yaml
deploy-job:
  script:
    - vault login -method=jwt role=ci jwt=$CI_JOB_JWT
    - export API_KEY=$(vault read -field=api_key secret/my-app)
    - ./deploy.sh --api-key $API_KEY
```

Popular options include:
- HashiCorp Vault
- AWS Secrets Manager
- Google Secret Manager
- Azure Key Vault

## Common Variable Use Cases

### Authentication Credentials

```yaml
deploy-job:
  script:
    - echo "Logging in to Docker registry..."
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

### Environment Configuration

```yaml
deploy-staging:
  variables:
    ENV_CONFIG: "staging"
  script:
    - echo "Deploying with $ENV_CONFIG configuration..."
    - cp config/$ENV_CONFIG.json config.json
  environment:
    name: staging

deploy-production:
  variables:
    ENV_CONFIG: "production"
  script:
    - echo "Deploying with $ENV_CONFIG configuration..."
    - cp config/$ENV_CONFIG.json config.json
  environment:
    name: production
```

### API Keys and Tokens

```yaml
integration-test:
  script:
    - echo "Testing API integration..."
    - curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com/test
```

## Practical Examples

### Complete Web Application Pipeline with Variables

```yaml
stages:
  - build
  - test
  - deploy

variables:
  APP_NAME: "my-web-app"
  IMAGE_NAME: $CI_REGISTRY_IMAGE/$APP_NAME

build-app:
  stage: build
  variables:
    NODE_ENV: "production"
  script:
    - echo "Building $APP_NAME in $NODE_ENV mode..."
    - npm ci
    - npm run build
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
  artifacts:
    paths:
      - dist/

test-app:
  stage: test
  variables:
    NODE_ENV: "test"
  script:
    - echo "Testing $APP_NAME..."
    - npm ci
    - npm test

.deploy-template: &deploy-template
  stage: deploy
  script:
    - echo "Deploying $APP_NAME to $CI_ENVIRONMENT_NAME..."
    - echo "Using DB connection: $DB_HOST:$DB_PORT/$DB_NAME"
    - kubectl set image deployment/$APP_NAME $APP_NAME=$IMAGE_NAME:$CI_COMMIT_SHA

deploy-staging:
  <<: *deploy-template
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-production:
  <<: *deploy-template
  environment:
    name: production
    url: https://example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

In this example:
- Global variables set project-wide values
- Job-specific variables override settings when needed
- Environment-scoped variables (set in GitLab UI) provide different database connections for staging vs production

## Best Practices

### Security Best Practices

- Never hardcode secrets in your pipeline configuration
- Always mask sensitive variables
- Use protected variables for production credentials
- Regularly rotate secrets and access tokens
- Provide minimal access permissions for service accounts
- Use environment-scoped variables to limit exposure

### Organizational Best Practices

- Use consistent naming conventions (e.g., uppercase for all variables)
- Document all variables and their purpose
- Group related variables with prefixes (e.g., `AWS_*` for AWS credentials)
- Regularly audit and clean up unused variables
- Prefer project variables over hardcoded values in `.gitlab-ci.yml`

### Technical Best Practices

- Keep variables focused on a single purpose
- Avoid setting too many variables at the global level
- Use group-level variables for shared configuration
- Use job-level variables for job-specific settings
- Test variables with echo commands before using them in critical operations

## Troubleshooting

### Common Issues

#### Variable Not Available

If a variable is not available in your job:

- Check the variable scope (project vs. group)
- Verify the variable is not protected or the job is on a protected branch
- Ensure the correct environment scope
- Check for typos in the variable name

#### Masked Variable Shows in Logs

If a masked variable is visible in logs:

- Verify the variable meets masking requirements
- Check if the variable is expanded inside another string
- Make sure the value doesn't match an environment variable

#### Job Fails Due to Missing Variable

If a job fails because a variable is missing:

- Add error handling to your scripts
- Provide default values for optional variables
- Use conditional checks for required variables

## Next Steps

Now that you understand variables and secrets in GitLab CI/CD, let's explore [Advanced CI/CD Techniques](09-advanced-techniques.md) to take your pipelines to the next level. 