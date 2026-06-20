# DevOps CI/CD Pipeline for a Node.js Application

## Purpose

This project demonstrates a complete CI/CD pipeline for a Node.js application, from source code to a running container on a live server, with no manual steps in between.

The scenario: a small team is building a Node.js app together. Multiple people are committing code, so there needs to be a way to guarantee that nobody's change breaks the app for everyone else, and that every approved change reaches a deployable Docker image automatically. That's what this pipeline does, it tests every change, and only builds and ships a Docker image if the tests pass.

This project demonstrates:
- Setting up version control for a multi-developer codebase
- Containerizing an application with Docker
- Building a Jenkins CI/CD pipeline with automated testing as a hard gate before deployment
- Versioning and tagging Docker images for traceability
- Deploying a containerized app to a cloud server
- Refactoring pipeline logic into a reusable Jenkins Shared Library

**Live repo:** [devops-bootcamp-node-project](https://github.com/elemasamuel/devops-bootcamp-node-project)
**Shared library:** [devops-bootcamp-jenkins-shared-library](https://github.com/elemasamuel/devops-bootcamp-jenkins-shared-library)

## Tech Stack

| Category | Tools |
|---|---|
| Application | Node.js |
| Version Control | Git, GitHub |
| Containerization | Docker, Docker Hub |
| CI/CD | Jenkins, Jenkins Shared Libraries |
| Deployment | DigitalOcean Droplet |
| Build Tooling | npm, Pipeline Utility Steps (Jenkins plugin) |

## Pipeline at a Glance

```
Developer Push → GitHub
       │
       ▼
   Jenkins Pipeline
   ├── 1. Bump version (npm version patch)
   ├── 2. Run automated tests (abort on failure)
   ├── 3. Build Docker image (tagged with new version)
   ├── 4. Push image to Docker Hub
   └── 5. Commit & push version bump to GitHub
       │
       ▼
   Manual Deploy → DigitalOcean Droplet (port 3000)
```

---

## Step 1: Set Up Version Control

**Purpose:** the app needed its own Git repository, separate from the original template, so the team could collaborate on it directly.

```sh
git clone https://gitlab.com/devops-bootcamp3/node-project.git
cd node-project

# remove remote repo reference
rm -rf .git

# create our own local repository and commit its content
git init
git add .
git commit -m "Initial commit"

# create the GitHub repo and point our local repo at it
git remote add origin git@github.com:elemasamuel/devops-bootcamp-node-project.git

# rename master to main (GitHub's default)
git branch -M main
git push -u origin main
```

![New GitHub repository with initial commit](./screenshots/01-github-repo-initial-commit.png)

---

## Step 2: Containerize the Application

**Purpose:** the app needed to run the same way on every machine — locally, in CI, and on the production server. Docker solves that.

```dockerfile
FROM node:13-alpine

RUN mkdir -p /usr/app
COPY app/images/* /usr/app/images/
COPY app/index.html /usr/app/
COPY app/package.json /usr/app/
COPY app/server*.js /usr/app/

WORKDIR /usr/app
EXPOSE 3000

RUN npm install

CMD ["node", "server.js"]
```

Committed and pushed the `Dockerfile` to the repo.

![Dockerfile committed to the repo](./screenshots/02-dockerfile-committed.png)
![Local docker build succeeding](./screenshots/03-docker-build-success.png)

---

## Step 3: Build the CI/CD Pipeline

**Purpose:** this is the core of the project. Every commit needs to automatically:
1. Bump the app's version
2. Run the test suite — and stop immediately if anything fails
3. Build a Docker image tagged with that version
4. Push the image to Docker Hub
5. Commit the version bump back to GitHub, so the repo always matches what's been shipped

**Setup required:** Jenkins needed Node and npm installed, plus stored credentials for GitHub and Docker Hub so the pipeline could authenticate without hardcoding secrets. I also installed the **Pipeline Utility Steps** plugin for its `readJSON` function, used to read the version back out of `package.json` after it's bumped.

![Jenkins credentials configured for GitHub and DockerHub](./screenshots/04-jenkins-credentials.png)
![Pipeline Utility Steps plugin installed](./screenshots/05-jenkins-plugin-installed.png)

### 3.1 — Bump Version

```groovy
stage('Bump Version') {
    steps {
        script {
            echo 'incrementing patch version...'
            dir('app') {
                sh 'npm version patch'

                def packageJson = readJSON file: 'package.json'
                def version = packageJson.version

                env.IMAGE_VERSION = "$version-$BUILD_NUMBER"
            }
        }
    }
}
```

`npm version patch` increments the version in `package.json`. `readJSON` reads it back so it can be combined with the Jenkins build number into a unique image tag (e.g. `1.0.2-11`).

### 3.2 — Run Tests

```groovy
stage('Run Tests') {
    steps {
        script {
            dir('app') {
                sh 'npm install'
                sh 'npm run test'
            }
        }
    }
}
```

This is the gate. If a test fails here, the pipeline stops — nothing gets built or pushed.

![Jenkins console output showing tests passing](./screenshots/06-jenkins-tests-passing.png)

### 3.3 — Build and Push Docker Image

```groovy
stage('Build and Push Docker Image') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            sh "docker build -t elemasamuel/fesi-repo:devops-bootcamp-node-project-${IMAGE_VERSION} ."
            sh "echo ${PASSWORD} | docker login -u ${USERNAME} --password-stdin"
            sh "docker push elemasamuel/fesi-repo:devops-bootcamp-node-project-${IMAGE_VERSION}"
        }
    }
}
```

Only runs if tests passed. Builds the image with the version tag from step 3.1, then logs in and pushes to Docker Hub using credentials stored in Jenkins, not in the code.

![New image tag visible on Docker Hub](./screenshots/07-dockerhub-new-tag.png)

### 3.4 — Commit Version Update

```groovy
stage('Commit Version Update') {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                sh 'git config user.email "jenkins@example.com"'
                sh 'git config user.name "jenkins"'

                sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@github.com/elemasamuel/devops-bootcamp-node-project.git"
                sh 'git add app/package.json'
                sh 'git commit -m "ci: version bump"'
                sh 'git push origin HEAD:main'

                sh 'git config --unset user.email'
                sh 'git config --unset user.name'
            }
        }
    }
}
```

The version bump from step 3.1 only existed inside the Jenkins workspace until this stage commits and pushes it back to GitHub. Without this, `package.json` in the repo would drift out of sync with what's actually been built and shipped.

![Full pipeline run, all stages green](./screenshots/08-jenkins-full-pipeline-green.png)
![Version-bump commit visible in GitHub history](./screenshots/09-github-version-bump-commit.png)

---

## Step 4: Deploy to a Server

**Purpose:** confirm the image the pipeline produced actually runs and serves traffic, not just that it built successfully.

```sh
docker login
# enter Docker Hub username and password

docker run -p3000:3000 -d elemasamuel/fesi-repo:devops-bootcamp-node-project-1.0.2-11
```

Opened port `3000` in the droplet's firewall, then loaded `http://<droplet-ip>:3000/` in a browser to confirm the app was live.

![Firewall rule opening port 3000](./screenshots/10-droplet-firewall-port-3000.png)
![Application running live in the browser](./screenshots/11-app-live-in-browser.png)

---

## Step 5: Extract a Reusable Jenkins Shared Library

**Purpose:** another team wanted the same pipeline logic for a different project. Copy-pasting the `Jenkinsfile` would mean any future fix has to be repeated manually in every project that copied it. A Jenkins Shared Library solves that — the logic lives in one place and any project can call it.

| Function | Purpose |
|---|---|
| `bumpNpmVersion(appDir, versionPart)` | Bumps the npm package version by the given part (`patch`/`minor`/`major`) |
| `runNpmTests(appDir)` | Installs dependencies and runs the test suite |
| `buildAndPublishImage(imageTag)` | Builds and pushes the Docker image |
| `commitAndPushVersionUpdate(gitRepo, credentialsId, branch)` | Commits and pushes the version bump to Git |

```groovy
// vars/bumpNpmVersion.groovy
def call(String appDir, String versionPart) {
    echo "incrementing ${versionPart} version..."
    dir("${appDir}") {
        sh "npm version ${versionPart}"

        def packageJson = readJSON file: 'package.json'
        def version = packageJson.version

        env.IMAGE_VERSION = "$version-$BUILD_NUMBER"
    }
}
```

```groovy
// vars/runNpmTests.groovy
def call(String appDir) {
    dir("${appDir}") {
        sh 'npm install'
        sh 'npm run test'
    }
}
```

```groovy
// vars/commitAndPushVersionUpdate.groovy
def call(String gitRepo, String credentialsId, String branch) {
    withCredentials([usernamePassword(credentialsId: "${credentialsId}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        sh 'git config user.email "jenkins@example.com"'
        sh 'git config user.name "jenkins"'

        sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@${gitRepo}"
        sh 'git add .'
        sh 'git commit -m "ci: version bump"'
        sh "git push origin HEAD:${branch}"

        sh 'git config --unset user.email'
        sh 'git config --unset user.name'
    }
}
```

With the library in place, the `Jenkinsfile` becomes a short list of calls:

```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    stages {
        stage('Bump Version') {
            steps { script { bumpNpmVersion('app', 'patch') } }
        }
        stage('Run Tests') {
            steps { script { runNpmTests('app') } }
        }
        stage('Build and Push Docker Image') {
            steps { script { buildAndPublishImage("elemasamuel/fesi-repo:devops-bootcamp-node-project-${IMAGE_VERSION}") } }
        }
        stage('Commit Version Update') {
            steps { script { commitAndPushVersionUpdate('github.com/elemasamuel/devops-bootcamp-node-project.git', 'GitHub', 'shared-library') } }
        }
    }
}
```

**Issue encountered:** I wanted `versionPart` to default to `'patch'` so callers could omit it. Jenkins shared libraries don't support Groovy default parameter values, it throws a script-approval error (`Scripts not permitted to use method ... invokeMethod ...`). Every call site passes all parameters explicitly instead.

![Jenkinsfile referencing the shared library](./screenshots/12-jenkinsfile-shared-library-calls.png)
![Shared library repo structure](./screenshots/13-shared-library-repo-structure.png)
![Pipeline running successfully using the shared library](./screenshots/14-shared-library-pipeline-green.png)

---

## What This Demonstrates

- A CI/CD pipeline where automated tests act as a real gate, not a formality — failing tests stop the build before anything reaches Docker Hub.
- Image versioning tied to both the app version and the Jenkins build number, so any image in the registry can be traced back to an exact commit.
- Credential handling through Jenkins' `withCredentials` instead of hardcoded secrets.
- Pipeline logic extracted into a Jenkins Shared Library, so it can be reused across projects instead of copy-pasted.

