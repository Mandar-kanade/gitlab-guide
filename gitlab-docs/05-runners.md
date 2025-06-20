# GitLab Runners

GitLab Runners are the agents that execute the jobs defined in your `.gitlab-ci.yml` file. This document covers everything you need to know about runners, from basic concepts to advanced configuration.

## What is a GitLab Runner?

A GitLab Runner is a lightweight application that processes jobs from a GitLab instance and reports results back to GitLab. Runners can be installed on various platforms and can run jobs in different environments.

## Types of Runners

### Shared Runners

Shared runners are available to all projects in a GitLab instance. They are maintained by the GitLab instance administrators.

Benefits:
- No setup required
- Available immediately
- Maintained by administrators

Limitations:
- Limited minutes on GitLab.com
- May have queues during peak times
- Limited customization options

### Specific Runners

Specific runners are dedicated to specific projects. You need to set them up yourself, but they offer more control.

Benefits:
- Custom environment
- Dedicated resources
- No queue time
- No minute restrictions

Limitations:
- Requires setup and maintenance
- Consumes your own resources

### Group Runners

Group runners are available to all projects in a specific group.

Benefits:
- Shared among related projects
- More efficient resource usage than specific runners
- Maintained by group administrators

## Runner Executors

Executors determine the environment where jobs run. GitLab Runner supports several executors:

### Shell Executor

Runs commands directly on the host machine.

```bash
# Registration example for shell executor
gitlab-runner register \
  --url "https://gitlab.com/" \
  --registration-token "PROJECT_REGISTRATION_TOKEN" \
  --description "Shell Runner" \
  --executor "shell"
```

**Pros:**
- Simple to set up
- Direct access to the system
- No overhead

**Cons:**
- Less isolation between jobs
- Dependency conflicts possible
- System dependencies required

**Best for:**
- Simple projects
- Trusted code
- System-specific operations

### Docker Executor

Runs each job in a fresh Docker container.

```bash
# Registration example for Docker executor
gitlab-runner register \
  --url "https://gitlab.com/" \
  --registration-token "PROJECT_REGISTRATION_TOKEN" \
  --description "Docker Runner" \
  --executor "docker" \
  --docker-image "alpine:latest"
```

**Pros:**
- Isolated environments
- Customizable containers
- Clean environment for each job

**Cons:**
- Requires Docker
- Slightly slower startup
- Persistent data needs volume mounts

**Best for:**
- Web applications
- Multiple language projects
- Projects requiring isolation

### Kubernetes Executor

Runs jobs as pods in a Kubernetes cluster.

```bash
# Registration example for Kubernetes executor
gitlab-runner register \
  --url "https://gitlab.com/" \
  --registration-token "PROJECT_REGISTRATION_TOKEN" \
  --description "Kubernetes Runner" \
  --executor "kubernetes" \
  --kubernetes-host "https://kubernetes.example.com" \
  --kubernetes-namespace "gitlab-runner"
```

**Pros:**
- Highly scalable
- Efficient resource usage
- Autoscaling capabilities

**Cons:**
- Complex setup
- Requires Kubernetes knowledge
- Higher overhead

**Best for:**
- Enterprise applications
- High-scale projects
- Cloud-native applications

### Other Executors

GitLab also supports other executors:
- **SSH**: Runs commands over SSH on a remote machine
- **VirtualBox**: Runs jobs in VirtualBox VMs
- **Parallels**: Runs jobs in Parallels VMs
- **Custom**: Build your own executor

## Installing and Configuring Runners

### Installation

#### Linux

```bash
# For Debian/Ubuntu
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# For RHEL/CentOS
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
sudo yum install gitlab-runner
```

#### macOS

```bash
# Using Homebrew
brew install gitlab-runner
brew services start gitlab-runner
```

#### Windows

```powershell
# Download the installer
$env:RUNNER_VERSION = "16.1.0"
Invoke-WebRequest -Uri "https://gitlab-runner-downloads.s3.amazonaws.com/v$env:RUNNER_VERSION/binaries/gitlab-runner-windows-amd64.exe" -OutFile "gitlab-runner.exe"

# Install and register
.\gitlab-runner.exe install
.\gitlab-runner.exe start
```

#### Docker

```bash
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

### Registration

To register a runner, you need a registration token from your GitLab instance:

1. Go to the project or group settings (or admin area for shared runners)
2. Navigate to CI/CD > Runners
3. Note the registration token
4. Run the registration command:

```bash
gitlab-runner register
```

You'll be prompted for:
- The GitLab instance URL
- The registration token
- A description for the runner
- Tags for the runner (optional)
- The executor type
- Executor-specific configuration

### Configuration File

GitLab Runner's configuration is stored in `/etc/gitlab-runner/config.toml` (or `~/.gitlab-runner/config.toml` for non-root installations). Here's an example configuration:

```toml
concurrent = 4
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "Docker Runner"
  url = "https://gitlab.com/"
  token = "TOKEN"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
```

### Common Configuration Options

#### Concurrency

The `concurrent` setting controls how many jobs can run simultaneously on a runner:

```toml
concurrent = 4  # Run up to 4 jobs at the same time
```

#### Job Timeout

You can set a global timeout for all jobs:

```toml
[[runners]]
  # ...
  executor = "docker"
  [runners.docker]
    # ...
  [runners.custom]
    job_timeout = 3600  # 1 hour in seconds
```

#### Cache Configuration

Configure where job caches are stored:

```toml
[[runners]]
  # ...
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      AccessKey = "ACCESSKEY"
      SecretKey = "SECRETKEY"
      BucketName = "runner-cache"
      BucketLocation = "us-east-1"
```

## Advanced Runner Features

### Autoscaling

For Docker Machine and Kubernetes executors, you can configure autoscaling:

```toml
[[runners]]
  # ...
  executor = "docker+machine"
  [runners.machine]
    IdleCount = 1
    IdleTime = 1800
    MaxBuilds = 100
    MachineDriver = "digitalocean"
    MachineName = "gitlab-docker-machine-%s"
    MachineOptions = [
      "digitalocean-image=ubuntu-20-04-x64",
      "digitalocean-region=nyc1",
      "digitalocean-size=s-1vcpu-1gb",
      "digitalocean-access-token=TOKEN"
    ]
```

### Runner Tags

Tags allow you to route jobs to specific runners:

```yaml
# In .gitlab-ci.yml
job:
  tags:
    - docker
    - linux
  script:
    - echo "This job runs only on runners with docker and linux tags"
```

```toml
# In config.toml
[[runners]]
  # ...
  tags = ["docker", "linux"]
```

### Runner Protected Status

Protected runners only execute jobs from protected branches or tags:

```toml
[[runners]]
  # ...
  protected = true
```

### Custom Environment Variables

You can define environment variables for all jobs run by a specific runner:

```toml
[[runners]]
  # ...
  environment = ["RUNNER_ENV_VAR=value", "ANOTHER_VAR=another_value"]
```

## Troubleshooting Runners

### Common Issues

#### Runner Not Picking Up Jobs

Check:
- Runner is registered and enabled
- Runner tags match job tags
- Runner is not in "paused" state
- Runner has not reached its job limit

#### Job Timeouts

Solutions:
- Increase timeout in job configuration
- Optimize job to run faster
- Split into smaller jobs

#### Docker-related Issues

Solutions:
- Verify Docker daemon is running
- Check image availability
- Adjust volume mounts and permissions

### Logs and Debugging

Runner logs can be found in:
- Linux: `/var/log/gitlab-runner/`
- macOS: `~/Library/Logs/gitlab-runner/`
- Windows: `C:\GitLab-Runner\logs\`

Increase log verbosity:

```bash
gitlab-runner --debug run
```

## Best Practices

### Security

- Use Docker's `privileged` mode only when necessary
- Avoid storing sensitive data in runner configurations
- Regularly update runners to patch security vulnerabilities
- Consider using ephemeral runners for sensitive projects

### Performance

- Use appropriate concurrency for your hardware
- Choose lightweight images when using Docker
- Implement efficient caching strategies
- Consider SSD storage for cache directories

### Maintenance

- Set up monitoring for your runners
- Implement automatic runner updates
- Document your runner configurations
- Regularly review runner usage and adjust resources

## Example: Full Runner Configuration

Here's a comprehensive example of a Docker runner configuration:

```toml
concurrent = 4
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "Docker Runner"
  url = "https://gitlab.com/"
  token = "TOKEN"
  executor = "docker"
  environment = ["DOCKER_DRIVER=overlay2", "CUSTOM_ENV_VAR=value"]
  [runners.custom_build_dir]
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      AccessKey = "ACCESSKEY"
      SecretKey = "SECRETKEY"
      BucketName = "runner-cache"
      BucketLocation = "us-east-1"
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    pull_policy = "if-not-present"
```

## Next Steps

Now that you understand GitLab Runners, let's explore [Artifacts and Dependencies](06-artifacts-dependencies.md) in more detail. 