# NodeJS CI/CD Pipeline with Jenkins

This is a step-by-step walkthrough of building a complete CI/CD pipeline for a NodeJS application using Jenkins, Docker, and a Jenkins Shared Library. By the end, every push to GitHub automatically triggers tests, a version bump, a Docker build and push, and a commit back to the repo, all without touching Jenkins manually.

## Tech stack

- **NodeJS** and **Express** for the application
- **Jest** for testing
- **Docker** for containerizing the app
- **Jenkins** for orchestrating the pipeline
- **Jenkins Shared Library** for reusable pipeline logic
- **GitHub webhooks** for automatic triggering on every push

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

---

## Step 0: Create the Git repository

Clone the starter project and set it up as your own repo.

```sh
git clone https://gitlab.com/devops-bootcamp3/node-project.git
cd node-project

# remove the original remote
rm -rf .git

# create a fresh local repo and commit everything
git init
git add .
git commit -m "Initial commit"

# point it at your own GitHub repo
git remote add origin git@github.com:elemasamuel/nodejs-cicd-jenkins-pipeline-project.git
git branch -M main
git push -u origin main
```

---

## Step 1: Dockerize the app

Create a `Dockerfile` in the project root:

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

Commit and push it:

```sh
git add Dockerfile
git commit -m "Add Dockerfile"
git push origin main
```

---

## Step 2: Set up Jenkins

This project runs Jenkins inside a Docker container on a DigitalOcean droplet, using the official `jenkins/jenkins:lts` image:

```sh
docker run -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home -d --name jenkins jenkins/jenkins:lts
```

A few things need to be installed and configured inside this container before any pipeline will work.

### Install Node and NPM inside the Jenkins container

The pipeline runs `npm` commands, so the Jenkins container itself needs Node installed (installing it on the host droplet doesn't help, since Jenkins is isolated inside its own container).

```sh
docker exec -u root -it jenkins bash
```

Then, inside the container:

```sh
apt-get update
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt-get install -y nodejs
```

Confirm it worked:

```sh
node -v
npm -v
```

### Confirm Docker access inside the Jenkins container

The pipeline also needs to run `docker build` and `docker push`. This requires the Docker CLI inside the container, plus access to the host's Docker socket (commonly mounted with `-v /var/run/docker.sock:/var/run/docker.sock` when starting the Jenkins container).

Check it works:

```sh
docker -v
docker ps
```

If `docker ps` shows your running containers (including Jenkins itself), the socket is mounted correctly.

Exit back out of the container shell:

```sh
exit
```

### Install the Pipeline Utility Steps plugin

This plugin provides the `readJSON` step, used to read the updated version out of `package.json` after bumping it.

In the Jenkins web UI:

1. **Manage Jenkins → Plugins → Available plugins**
2. Search for **Pipeline Utility Steps**
3. Install it

### Add GitHub credentials

GitHub no longer accepts account passwords for git operations, so a Personal Access Token is needed instead.

1. On GitHub: **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**
2. Give it a name, set an expiration, and check the **repo** scope
3. Generate it and copy the token immediately, it's only shown once

In Jenkins:

1. **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. **Kind**: Username with password
3. **Username**: your GitHub username
4. **Password**: the token you just generated
5. **ID**: `GitHub`
6. Save

### Add DockerHub credentials

1. On DockerHub: **Account Settings → Security → New Access Token**
2. Give it Read & Write permissions, generate it, and copy the token

In Jenkins:

1. **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. **Kind**: Username with password
3. **Username**: your DockerHub username
4. **Password**: the access token
5. **ID**: `DockerHub`
6. Save

---

## Step 3: Build the pipeline

### Add a stage for bumping the version

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

This bumps the patch version in `app/package.json` and stores the new version, combined with the Jenkins build number, in `env.IMAGE_VERSION`. Because `env` variables are shared across the whole pipeline run, later stages can reference `${IMAGE_VERSION}` directly.

### Add a stage for running tests

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

If the tests fail, this stage fails and the pipeline stops here, nothing gets built or pushed.

### Add a stage for building and pushing the Docker image

```groovy
stage('Build and Push Docker Image') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            sh "docker build -t elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-${IMAGE_VERSION} ."
            sh "echo ${PASSWORD} | docker login -u ${USERNAME} --password-stdin"
            sh "docker push elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-${IMAGE_VERSION}"
        }
    }
}
```

### Add a stage for committing the version bump

```groovy
stage('Commit Version Update') {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                sh 'git config user.email "jenkins@example.com"'
                sh 'git config user.name "jenkins"'

                sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@github.com/elemasamuel/nodejs-cicd-jenkins-pipeline-project.git"
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

### Putting it all together

The full `Jenkinsfile` at this point:

```groovy
pipeline {
    agent any
    stages {
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
        stage('Build and Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh "docker build -t elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-${IMAGE_VERSION} ."
                    sh "echo ${PASSWORD} | docker login -u ${USERNAME} --password-stdin"
                    sh "docker push elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-${IMAGE_VERSION}"
                }
            }
        }
        stage('Commit Version Update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh 'git config user.email "jenkins@example.com"'
                        sh 'git config user.name "jenkins"'

                        sh "git remote set-url origin https://${USERNAME}:${PASSWORD}@github.com/elemasamuel/nodejs-cicd-jenkins-pipeline-project.git"
                        sh 'git add app/package.json'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:main'

                        sh 'git config --unset user.email'
                        sh 'git config --unset user.name'
                    }
                }
            }
        }
    }
}
```

Commit and push the Jenkinsfile, then create the pipeline job in Jenkins:

1. Jenkins dashboard → **New Item**
2. Name it, select **Pipeline**, click OK
3. Under **Pipeline**, set Definition to **Pipeline script from SCM**
4. **SCM**: Git
5. **Repository URL**: your repo's HTTPS URL
6. **Credentials**: select `GitHub`
7. **Branch Specifier**: `*/main`
8. **Script Path**: `Jenkinsfile`
9. Save, then click **Build Now**

If everything is wired up correctly, the build runs through all four stages, bumping the version, running tests, building and pushing the image, and committing the version bump back to GitHub.

---

## Step 4: Manually deploy the image

Once the pipeline has pushed a new image to DockerHub, it can be deployed to a server.

SSH into the server and run:

```sh
docker login
# enter DockerHub username and password/token when prompted

docker run -p 3000:3000 -d elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-<version>
```

If the server has a cloud firewall (DigitalOcean droplets often do), open port 3000 for inbound TCP traffic, otherwise the app will be unreachable from outside even though the container is running.

Test it by opening `http://<server-ip>:3000` in a browser.

---

## Step 5: Extract the pipeline logic into a Jenkins Shared Library

As the pipeline grows, or as more projects need the same kind of pipeline, copying the same Groovy code into every Jenkinsfile becomes a maintenance problem. A Jenkins Shared Library solves this by moving the reusable logic into its own repository, so any Jenkinsfile can call into it with a single line.

### Create the shared library repository

Create a new, separate GitHub repo, for example `jenkins-shared-library`, and clone it locally:

```sh
git clone https://github.com/elemasamuel/jenkins-shared-library.git
cd jenkins-shared-library
mkdir vars
```

Jenkins shared libraries expect a `vars/` folder. Each `.groovy` file inside it becomes a callable function, named after the file.

### Add the version bump function

```sh
cat > vars/bumpNpmVersion.groovy << 'EOF'
#!/usr/bin/env groovy
def call(String appDir, String versionPart) {
    echo "incrementing ${versionPart} version..."
    dir("${appDir}") {
        sh "npm version ${versionPart}"

        def packageJson = readJSON file: 'package.json'
        def version = packageJson.version

        env.IMAGE_VERSION = "$version-$BUILD_NUMBER"
    }
}
EOF
```

### Add the test runner function

```sh
cat > vars/runNpmTests.groovy << 'EOF'
#!/usr/bin/env groovy
def call(String appDir) {
    dir("${appDir}") {
        sh 'npm install'
        sh 'npm run test'
    }
}
EOF
```

### Add the Docker build and push function

```sh
cat > vars/buildAndPublishImage.groovy << 'EOF'
#!/usr/bin/env groovy
def call(String imageName) {
    withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        sh "docker build -t ${imageName} ."
        sh "echo ${PASSWORD} | docker login -u ${USERNAME} --password-stdin"
        sh "docker push ${imageName}"
    }
}
EOF
```

### Add the commit and push function

```sh
cat > vars/commitAndPushVersionUpdate.groovy << 'EOF'
#!/usr/bin/env groovy
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
EOF
```

Commit and push the library:

```sh
git add .
git commit -m "Add shared library functions"
git push origin main
```

### Register the library in Jenkins

1. **Manage Jenkins → System → Global Pipeline Libraries → Add**
2. **Name**: `jenkins-shared-library` (this exact name is what Jenkinsfiles will reference)
3. **Default version**: `main`
4. **Retrieval method**: Modern SCM → Git
5. **Project Repository**: the shared library's HTTPS URL
6. **Credentials**: `GitHub`
7. Save

### Update the Jenkinsfile to use the shared library

Replace the node project's `Jenkinsfile` with:

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

The `library('jenkins-shared-library')` line tells Jenkins to fetch the shared library before running the pipeline, after which every function inside its `vars/` folder is available to call directly, as if it were built into Jenkins.

Commit, push, and run the pipeline again to confirm it still works the same way as before, just with far less code in the Jenkinsfile itself.

---

## Step 6: Switch to a Multibranch Pipeline

A regular Pipeline job only watches one branch. A Multibranch Pipeline job scans the entire repository, automatically creating a pipeline for every branch that contains a Jenkinsfile, and removing it when the branch is deleted. This matters once development happens across multiple feature branches rather than just `main`.

1. Jenkins dashboard → **New Item**
2. Name it, select **Multibranch Pipeline**, click OK
3. Under **Branch Sources**, click **Add source**, choose **GitHub** (or plain Git)
4. **Credentials**: `GitHub`
5. **Repository HTTPS URL**: the node project's repo URL
6. Leave the default Behaviors as is
7. Under **Build Configuration**, set Mode to **by Jenkinsfile** and Script Path to `Jenkinsfile`
8. Save

Jenkins immediately scans the repository and creates a pipeline for each branch with a Jenkinsfile.

---

## Step 7: Automatic triggering with a GitHub webhook

Without a webhook, the pipeline only runs when someone clicks **Build Now**. A webhook makes GitHub notify Jenkins the instant a push happens, triggering the pipeline automatically.

The GitHub plugin in Jenkins exposes a fixed listener path for this, `/github-webhook/`, at whatever host and port Jenkins runs on.

1. On the GitHub repo: **Settings → Webhooks → Add webhook**
2. **Payload URL**: `http://<jenkins-host>:8080/github-webhook/`
3. **Content type**: `application/json`
4. **Which events**: Just the push event
5. **Active**: checked
6. **Add webhook**

GitHub sends a test ping immediately, shown with a green checkmark if Jenkins responded successfully.

From this point on, every `git push` to the repository triggers the pipeline automatically, no manual builds needed.

---

## Verifying everything works end to end

1. Make a small change and push it
2. Watch the Multibranch Pipeline job in Jenkins, a new build should start within seconds, without clicking Build Now
3. Confirm the build bumps the version, passes the tests, pushes a new Docker image, and commits the version bump back to GitHub

## What's next

- Add a deployment stage that automatically deploys new images to a live server after a successful build
- Restrict automatic deployment to only the `main` branch, while still running tests on feature branches
