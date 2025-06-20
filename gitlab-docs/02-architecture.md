# GitLab CI/CD Architecture

GitLab CI/CD architecture consists of two main components: **GitLab server** and **GitLab Runner**. Understanding how these components interact is essential for effectively using GitLab CI/CD.

## Core Components

### GitLab Server

The GitLab server is responsible for:

- Managing CI/CD pipelines and the associated jobs
- Storing and serving artifacts
- Saving results and logs of pipeline executions
- Coordinating the work and delegating execution to runners
- Providing a user interface for monitoring and configuration

The server maintains the state of all pipelines and makes decisions about which jobs should be executed next.

### GitLab Runner

GitLab Runners are the workhorses of the CI/CD system. They:

- Execute the jobs defined in the pipeline
- Receive commands from the GitLab server
- Run the scripts defined in the jobs
- Report results back to the GitLab server

Runners can be installed on various platforms and can be shared across multiple projects or dedicated to specific projects.

## Execution Flow

When a pipeline is triggered, the following flow occurs:

1. A change is pushed to the GitLab repository
2. GitLab detects the change and looks for a `.gitlab-ci.yml` file
3. The GitLab server parses the `.gitlab-ci.yml` file and creates a pipeline
4. The server queues jobs that should be executed in the current stage
5. Available runners pick up jobs from the queue
6. Runners:
   - Pull the specified Docker image (if using Docker executor)
   - Clone the repository code
   - Fetch any required artifacts from previous stages
   - Execute the job's script
   - Upload artifacts to the GitLab server (if configured)
   - Return execution status to the server (success or failure)
7. GitLab server processes the results and queues jobs from the next stage if all jobs in the current stage succeed

## Pipeline Execution Diagram

```
┌─────────────┐           ┌─────────────┐           ┌─────────────┐
│             │           │             │           │             │
│  Code Push  │──────────►│  GitLab     │◄─────────►│  GitLab     │
│             │           │  Server     │           │  Runner     │
└─────────────┘           └─────────────┘           └─────────────┘
                               │  ▲                       │  ▲
                               ▼  │                       ▼  │
                          ┌─────────────┐           ┌─────────────┐
                          │             │           │             │
                          │  Pipeline   │           │  Job        │
                          │  Creation   │           │  Execution  │
                          │             │           │             │
                          └─────────────┘           └─────────────┘
```

## Runner Executors

GitLab Runners use executors to define the environment where jobs run. Common executor types include:

- **Shell**: Runs jobs in the system shell on the machine where the runner is installed
- **Docker**: Uses Docker containers to provide isolated execution environments
- **Kubernetes**: Runs jobs as Kubernetes pods in a cluster
- **Virtual Machine**: Creates a VM for each job
- **SSH**: Connects to a remote machine via SSH to run jobs

Docker is the most commonly used executor as it provides a good balance between isolation, performance, and ease of use.

## Types of Runners

### Shared Runners

- Provided by GitLab.com or your GitLab instance administrator
- Available to all projects
- Suitable for general-purpose CI/CD tasks
- May have usage limits (minutes per month)

### Specific Runners

- Dedicated to specific projects
- Provide more control over the execution environment
- Can be configured with special hardware or software requirements
- Often self-hosted on your own infrastructure

### Group Runners

- Available to all projects in a specific group
- Provide a balance between shared and specific runners

## Self-Hosted vs. GitLab.com Runners

### GitLab.com Runners

- Provided by GitLab
- Limited free CI/CD minutes
- Easy to use with no setup required
- May have queue times during peak usage

### Self-Hosted Runners

- Installed and maintained on your infrastructure
- No limit on usage minutes
- Can be customized to your specific needs
- Require setup and maintenance

## Runner Registration

To use a custom runner, it needs to be registered with your GitLab instance:

1. Install the GitLab Runner binary on your machine
2. Register the runner using a registration token
3. Configure the runner's executor and options
4. Start the runner service

## Example: Runner Registration

```bash
# Install GitLab Runner
# For Linux:
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Register the runner
sudo gitlab-runner register \
  --url "https://gitlab.com/" \
  --registration-token "YOUR_REGISTRATION_TOKEN" \
  --description "My Docker Runner" \
  --executor "docker" \
  --docker-image "alpine:latest"

# Start the runner
sudo gitlab-runner start
```

## Next Steps

Now that you understand GitLab CI/CD architecture, let's look at [Pipeline Configuration Basics](03-pipeline-basics.md) to learn how to set up your own CI/CD pipeline. 