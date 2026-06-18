# Azure Pipeline

```
Developer Pushes Code
          │
          ▼
    Build Pipeline
          │
          ▼
      Artifact
          │
          ▼
   Release Pipeline
          │
          ▼
     Environment
 (AWS, Azure, VM, etc.)
```

## Build Pipeline

- A build pipeline takes code, builds it, tests it, and creates an artifact.

- A build can be used along with branch protection to ensure that specific quality checks are made before code is merged into a branch.

- A build pipeline is the first step towards continuous integration and continuous testing.

## Release Pipeline

- Release pipelines take the artifact and deploys it.

- A release pipeline is generally technology agnostic relative to the build artifact (e.g. .NET Core Web App, NodeJS, etc.) and the target environment (e.g. Azure, AWS, Virtual machine, etc)

- Needed for continuous delivery

---

## Azure DevOps Pipelines

- **Trigger** - tells a Pipeline to run.
- **Pipeline** - is made up of one or more stages. Can deploy to one or more targets.
- **Stage** - organizes jobs. Each stage can have one or more jobs.
- **Job** - runs on one agent. A job can also be agentless.
- **Agent** - runs a job that contains one or more steps.
- **Step** - a task that is executed as the smallest building block of a pipeline.
- **Task** - a pre-packaged script that performs an action like running unit tests or packaging a repo as a build artifact.
- **Artifact** - a collection of files or packages published by a pipeline run.

---

## Steps to Create a Pipeline

Azure devops pipelines are brnach based. So, the first step is to create a branch in your repo that contains the code you want to build and deploy.

1. Create a branch in your repo (e.g., "feature/new-pipeline").
2. Create a YAML file (e.g., "azure-pipelines.yml") in the root of your repo that defines the pipeline.
3. Commit and push the YAML file to the branch you created.
4. Azure DevOps will not automatically create a pipeline like in gthub actions. You need to go to Azure DevOps, navigate to Pipelines, and click "New Pipeline". Then, select your repo and choose the option to use an existing YAML file. Select the YAML file you created and follow the prompts to create the pipeline.

---

## Pipeline Agent

- An Agent is simply a machine that executes pipeline jobs.
- We can use Microsoft-hosted agents or self-hosted agents.

## Components of a Pipeline

<image src="./images/image.png" width="700px" />

- **Stage**: A major milestone in the pipeline (e.g., "Build", "QA Testing", "Production Deployment").
- **Job**: A collection of steps that execution runs on a single build agent. Jobs can run sequentially or in parallel.
- **Step / Task**: The smallest execution element within a job. A job contains a sequential list of tasks that do the heavy lifting.

each job runs on its own agent (VM) unless you specifically use a self-hosted agent and control where jobs run.

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        pool:
          vmImage: ubuntu-latest

  - stage: Test
    jobs:
      - job: TestJob
        pool:
          vmImage: ubuntu-latest
```

Even though both use ubuntu-latest, they are usually different fresh VMs.

### Task vs Script

- **Task**: A pre-packaged script that performs an action like installing runtime, running unit tests, or packaging a repo as a build artifact. Tasks are reusable and can be shared across pipelines.
- **Script**: A custom script that you write to perform a specific action. Scripts can be written in various scripting languages like PowerShell, Bash, or Python.

```yaml
trigger:
  - frontend

pool:
  vmImage: ubuntu-latest

steps:
  - task: UseNode@1
    displayName: "Use Node.js"
    inputs:
      version: "22.x"

  - script: |
      npm install
      npm run build
    displayName: "Install dependencies and build"

  - task: Docker@2
    displayName: "Build and Push Docker Image"
    inputs:
      containerRegistry: "kavindu_dockerhub"
      repository: "kavinduorg/azuredevops-frontend"
      command: "buildAndPush"
      Dockerfile: "./Dockerfile"
```

```yaml
trigger:
  - frontend

variables:
  NODE_VERSION: "22.x"
  IMAGE_NAME: "kavinduorg/azuredevops-frontend"
  DOCKER_REGISTRY_SERVICE_CONNECTION: "kavindu_dockerhub"

stages:
  - stage: Build
    displayName: "Build React Application"

    jobs:
      - job: BuildReact
        displayName: "Install, Test and Build"

        pool:
          vmImage: ubuntu-latest

        steps:
          - task: UseNode@1
            displayName: "Install Node.js"
            inputs:
              version: $(NODE_VERSION)

          - task: Cache@2
            displayName: "Cache NPM Packages"
            inputs:
              key: 'npm | "$(Agent.OS)" | package-lock.json'
              path: $(Pipeline.Workspace)/.npm
              restoreKeys: |
                npm | "$(Agent.OS)"

          - script: npm ci
            displayName: "Install Dependencies"

          - script: npm run lint
            displayName: "Run Lint"
            continueOnError: true

          - script: npm test -- --run
            displayName: "Run Tests"
            continueOnError: true

          - script: npm run build
            displayName: "Build React App"

          - task: PublishPipelineArtifact@1
            displayName: "Publish Build Artifact"
            inputs:
              targetPath: dist
              artifact: react-build

  - stage: Docker
    displayName: "Docker Build & Push"
    dependsOn: Build
    condition: succeeded()

    jobs:
      - job: DockerBuild
        displayName: "Build and Push Image"

        pool:
          vmImage: ubuntu-latest

        steps:
          - checkout: self

          - task: DownloadPipelineArtifact@2
            displayName: "Download Build Artifact"
            inputs:
              artifact: react-build
              path: dist

          - task: Docker@2
            displayName: "Build and Push Docker Image"
            inputs:
              containerRegistry: $(DOCKER_REGISTRY_SERVICE_CONNECTION)
              repository: $(IMAGE_NAME)
              command: buildAndPush
              Dockerfile: Dockerfile
              tags: |
                $(Build.BuildId)
                latest

  - stage: Deploy
    displayName: "Deploy"
    dependsOn: Docker
    condition: succeeded()

    jobs:
      - deployment: DeployToDev
        displayName: "Deploy to Development"
        environment: development

        strategy:
          runOnce:
            deploy:
              steps:
                - script: |
                    echo "Deploying image $(Build.BuildId)"
                displayName: "Deploy Application"
```

---

## Creating secrets and variables

### 1. Adding Regular (Non-Secret) Variables

#### Option A: Directly in the YAML File

This is the best approach for non-sensitive values like environment names, build configurations, or public API endpoints.

```yaml
variables:
  buildConfiguration: "Release"
  environmentName: "Production"

steps:
  - script: echo "Building in $(buildConfiguration) mode for $(environmentName)."
```

#### Option B: Via the Pipeline Web UI

- Go to Pipelines, select your pipeline, and click Edit.
- Click the Variables button in the top right corner.
- Click New variable, enter the Name and Value, and click OK.
- Consume it in your YAML file using the macro syntax: $(yourVariableName)

### 2. Adding Secret Variables

Secret variables are encrypted at rest. For security reasons, Azure DevOps will not let you define secrets directly as plain text inside YAML files. Instead, you must use one of the following secure options:

- Using the Pipeline Pipeline UI (Best for Pipeline-Specific Secrets)Navigate to your pipeline, click Edit, and select Variables.
- Click New variable (or add a new row).Enter the name and sensitive value.Check the box that says Keep this value secret.
- A lock icon will appear.Click Save. Once saved, the plain-text value is completely hidden and cannot be viewed in the UI again.

---

## Service Connections

Service connections are used to securely connect Azure DevOps to external services like Azure, AWS, Docker registries, etc. They allow your pipelines to authenticate and interact with these services without exposing sensitive credentials in your YAML files.
