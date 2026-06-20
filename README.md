# NodeJS CI/CD Pipeline with Jenkins

A complete CI/CD pipeline for a NodeJS application, built with Jenkins, Docker, and a reusable shared library. Every push to GitHub automatically triggers tests, a version bump, a Docker image build and push, and a commit back to the repo, with zero manual steps.

## What this project does

When code is pushed to this repository, Jenkins automatically:

1. Bumps the patch version of the app (`package.json`)
2. Installs dependencies and runs the test suite
3. Builds a Docker image tagged with the new version
4. Pushes that image to DockerHub
5. Commits the version bump back to GitHub

If the tests fail, the pipeline stops immediately and nothing gets built or pushed. This means only working, tested code ever reaches DockerHub.

## Tech stack

- **NodeJS** and **Express** for the application
- **Jest** for testing
- **Docker** for containerizing the app
- **Jenkins** for orchestrating the pipeline, running in a Docker container on a DigitalOcean droplet
- **Jenkins Shared Library** for reusable pipeline logic across projects
- **GitHub webhooks** for instant, automatic pipeline triggering on every push

## Project structure

```
.
├── app/
│   ├── images/
│   ├── index.html
│   ├── server.js
│   ├── server.test.js
│   └── package.json
├── Dockerfile
├── Jenkinsfile
└── README.md
```

## How the pipeline works

### The Jenkinsfile

The Jenkinsfile doesn't contain the pipeline logic directly. Instead, it pulls in a separate [Jenkins Shared Library](https://github.com/elemasamuel/jenkins-shared-library) and calls functions from it:

```groovy
#!/usr/bin/env groovy

library('jenkins-shared-library')

pipeline {
    agent any
    stages {
        stage('Bump Version') {
            steps {
                script {
                    bumpNpmVersion('app', 'patch')
                }
            }
        }
        stage('Run Tests') {
            steps {
                script {
                    runNpmTests('app')
                }
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    buildAndPublishImage("elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-${IMAGE_VERSION}")
                }
            }
        }
        stage('Commit Version Update') {
            steps {
                script {
                    commitAndPushVersionUpdate('github.com/elemasamuel/nodejs-cicd-jenkins-pipeline-project.git', 'GitHub', 'main')
                }
            }
        }
    }
}
```

### Why a shared library

Keeping the pipeline logic in a separate repo means any other NodeJS project can reuse the exact same stages just by writing a short Jenkinsfile like the one above. Fix a bug in the shared library once, and every project using it benefits immediately, no copy-pasting Groovy code between repos.

The shared library lives at [`jenkins-shared-library`](https://github.com/elemasamuel/jenkins-shared-library) and exposes four functions, one per file in its `vars/` folder:

| Function | What it does |
|---|---|
| `bumpNpmVersion(appDir, versionPart)` | Bumps the app version and stores it in `env.IMAGE_VERSION` for later stages to use |
| `runNpmTests(appDir)` | Installs dependencies and runs the test suite |
| `buildAndPublishImage(imageName)` | Builds the Docker image and pushes it to DockerHub |
| `commitAndPushVersionUpdate(gitRepo, credentialsId, branch)` | Commits the version bump and pushes it back to GitHub |

### Multibranch pipeline

Jenkins is configured as a **Multibranch Pipeline** job rather than a single-branch one. This means Jenkins automatically scans the whole repository, finds every branch that contains a Jenkinsfile, and builds a pipeline for each one on its own, no manual job creation needed when a new feature branch is created.

### Automatic triggering

A GitHub webhook is configured on this repository, pointing at the Jenkins server. Every push sends an instant notification to Jenkins, which kicks off the pipeline within seconds. There's no polling and no manual "Build Now" clicking involved.

## Running the app locally

Pull the latest image from DockerHub and run it:

```sh
docker pull elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-<version>
docker run -p 3000:3000 -d elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-<version>
```

Then open `http://localhost:3000` in your browser.

## Jenkins setup notes

A few setup details that aren't obvious from the Jenkinsfile alone, useful if reproducing this on a fresh Jenkins instance:

- **Node and Docker inside Jenkins**: since Jenkins runs in its own Docker container, the container needs Node, NPM, and the Docker CLI installed inside it, plus access to the host's Docker socket so it can build and push images.
- **Pipeline Utility Steps plugin**: required for the `readJSON` step used to read the updated version out of `package.json`.
- **Credentials**: two credentials are configured in Jenkins (Manage Jenkins → Credentials), both as "Username with password":
  - `GitHub`, using a GitHub Personal Access Token
  - `DockerHub`, using a DockerHub access token
- **Shared library registration**: registered under Manage Jenkins → System → Global Pipeline Libraries, pointing at the shared library repo on the `main` branch.
- **Webhook**: configured on the GitHub repo under Settings → Webhooks, pointing at `http://<jenkins-host>:8080/github-webhook/`.

## What's next

- Add a deployment stage to automatically deploy new images to a live server after a successful build
- Add a staging/production branch distinction so only `main` triggers a deploy
