## Exercises
<br />

Your team members want to collaborate on your NodeJS application, where you list developers with their projects. So they ask you to set up a git repository for it.

Also, you think it's a good idea to add tests, to test that no one accidentally breaks the existing code.

Moreover, you all decide every change should be immediately built and pushed to the Docker repository, so everyone can access it right away.

For that they ask you to **set up a continuous integration pipeline**.

![CI/CD Pipeline Workflow](https://github.com/user-attachments/assets/dc866c33-719e-4b6e-b8fd-a766a090f178)

<details>
<summary>Exercise 0: Create Git Repository</summary>
<br />

**Tasks:**

- clone the git repository `https://gitlab.com/devops-bootcamp3/node-project.git`
- create your own project/git repo from it

**Steps to solve the tasks:**

```sh
git clone https://gitlab.com/devops-bootcamp3/node-project.git
cd node-project

# remove remote repo reference
rm -rf .git
# create your own local repository and commit its content
git init 
git add .
git commit -m "Initial commit"

# create git repository on GitHub push your newly created local repository to it
git remote add origin git@github.com:elemasamuel/devops-bootcamp-node-project.git
# rename master branch of original Gitlab repository to main (default on GitHub)
git branch -M main
# push your newly created local repository to it
git push -u origin main
```
![Git push command output in terminal](https://github.com/user-attachments/assets/18cc2f6b-bae8-4511-9737-ee20fe3ae7f9)

</details>

******

<details>
<summary>Exercise 1: Dockerize your NodeJS App</summary>
<br />

**Tasks:**

Configure your application to be built as a Docker image.
- Dockerize your NodeJS app

**Steps to solve the tasks:**\
Step 1: Create a Dockerfile with the following content in the project root:
```sh
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

Commit and push the Dockerfile.

![Docker image build](https://github.com/user-attachments/assets/9bd6c69e-90f7-4812-88ca-d6a48fda7462)

</details>

******

<details>
<summary>Exercise 2: Create a full pipeline for your NodeJS App</summary>
<br />

**Tasks:**

You want the following steps to be included in your pipeline:
- Increment version\
  The application's version and docker image version should be incremented.
- Run tests\
  You want to test the code, to be sure to deploy only working code. When tests fail, the pipeline should abort.
- Build docker image with incremented version
- Push to Docker repository
- Commit to Git\
  The application version increment must be committed and pushed to a remote Git repository.

**Steps to solve the tasks:**
Step 1: Prerequisites (tools and credentials)\
For the build pipeline we need Node and NPM to be installed in the Docker container running Jenkins. We further need credentials for accessing GitHub and DockerHub. 

### Install Jenkins as a Docker container

See the [Jenkins Documentation](https://www.jenkins.io/doc/book/installing/docker/).

**Create a Droplet on DigitalOcean**

Login to your DigitalOcean account an create a new Droplet (4GB RAM). Jenkins needs at least 1GB RAM. Change the Droplet's name to something like 'jenkins-server' and attach a firewall rule-set to it opening port 22 for SSH from your machine's IP address and port 8080 (Type=Custom) for Jenkins from all IP addresses.

SSH into the Droplet (`ssh root@<droplet-ip>`) and install Docker (since we want to run Jenkins in a Docker container):
```sh
apt update
apt install docker.io
```

Start Jenkins in a Docker container:
```sh
docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
# Jenkins master and worker nodes communicate over port 50000
```

Copy the initial administrator password:
```sh
# in the docker container
docker exec -it <container-id> bash
cat /var/jenkins_home/secrets/initialAdminPassword

# or directly on the host
docker volume inspect jenkins_home
# the path is displayed as "Mountpoint" -> /var/lib/docker/volumes/jenkins_home/_data
cat /var/lib/docker/volumes/jenkins_home/_data/secrets/initialAdminPassword
```

Open Jenkins in the browser under `http://<droplet-ip>:8080`, enter the initial administrator password and install the suggested plugins. After creating a first admin user, the initialization of Jenkins is done and the welcome screen is displayed.

<img width="1897" height="956" alt="image" src="https://github.com/user-attachments/assets/447d9e62-f43d-4cf3-be85-43b6230dff45" />

To read the updated version from the package.json file, we need JSON support. That's why we install the "Pipeline Utility Steps" plugin. It provieds a `readJSON` function.

Now we can start writing the Jenkinsfile.

Step 2: Add a stage for incrementing the application version\
The patch version is incremented using the command `npm version patch`. To read the updated version from the package.json file, we use the `readJSON` function provided by the "Pipeline Utility Steps" plugin:
```groovy
#!/usr/bin/env groovy

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
    }
}
```

Step 3: Add a stage for running the tests\
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

Step 4: Add a stage for building and pushing the Docker image\
```groovy
stage('Build and Push Docker Image') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]){
            sh "docker build -t elemasamuel/fesi-repo:devops-bootcamp-node-project-${IMAGE_VERSION} ."
            sh "echo ${PASSWORD} | docker login -u ${USERNAME} --password-stdin"
            sh "docker push elemasamuel/fesi-repo:devops-bootcamp-node-project-${IMAGE_VERSION}"
        }
    }
}
```

Step 5: Add a stage for committing the package.json file with the incremented version\
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

</details>

******

<details>
<summary>Exercise 3: Manually deploy new Docker Image on server</summary>
<br />

**Tasks:**

After the pipeline has run successfully, you:
- Manually deploy the new docker image on the droplet server.

**Steps to solve the tasks:**
Step 1: ssh into a DigitalOcean droplet

Step 2: Execute the following commands:
```sh
docker login
# enter username and password for Docker-Hub

docker run -p3000:3000 -d elemasamuel/fesi-repo:devops-bootcamp-node-project-1.0.2-11
```

Step 3: Open port 3000\
Configure a firewall opening the port 3000 for all IP addresses.

Step 4: Test the application\
Open a browser and enter the URL `http://<droplet-ip>:3000/`.

</details>

******

<details>
<summary>Exercise 4: Extract into Jenkins Shared Library</summary>
<br />

**Tasks:**

A colleague from another project tells you, they are building a similar Jenkins pipeline and they could use some of your logic. So you suggest creating a Jenkins Shared Library to make your Jenkinsfile code reusable and shareable.

Therefore, you do the following:
- Extract all logic into Jenkins-shared-library with parameters and reference it in Jenkinsfile.

**Steps to solve the tasks:**
We can reuse and extend the Jenkins Shared Library on [GitHub](https://github.com/elemasamuel/devops-bootcamp-jenkins-shared-library.git) created in [demo project 3](./demo-projects/3-create-shared-library/).

Step 1: Extract code for the 'Bump Version' stage\
To make the logic reusable for other node projects, we pass in two parameters:
- the name of the directory containing the packaje.json file
- the version-part of be incremented ('patch', 'minor' or 'major')

**Note:** It would be nice to make the second parameter be an optional one with default value `patch`. But this results in an error during the build saying
```
Scripts not permitted to use method groovy.lang.GroovyObject invokeMethod java.lang.String java.lang.Object (org.jenkinsci.plugins.workflow.cps.CpsClosure2 bumpNpmVersion java.lang.String). Administrators can decide whether to approve or reject this signature.
```
It seems that optional parameters are a Groovy feature not allowed in Jenkins shared libraries.

Switch to the Shared Library project and create a new file called `bumpNpmVersion.groovy` in the `vars` folder with the following content:
```groovy
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
```

Replace the content of the `script` block in the 'Bump Version' stage with `bumpNpmVersion('app', 'patch')`.

Step 2: Extract code for the 'Run Tests' stage\
Create a new file called `runNpmTests.groovy` in the `vars` folder with the following content:
```groovy
#!/usr/bin/env groovy
def call(String appDir) {
    dir("${appDir}") {
        sh 'npm install'
        sh 'npm run test'
    } 
}
```

Replace the content of the `script` block in the 'Run Tests' stage with `runNpmTests('app')`.

Step 3: Reuse shared library code for the 'Build and Push Docker Image' stage\
The shared library function `buildAndPublishImage` we created for the demo project 3 can be reused as is. Just replace the content of the `script` block in the 'Build and Push Docker Image' stage with `buildAndPublishImage("elemasamuel/fesi-repo:devops-bootcamp-node-project-${IMAGE_VERSION}")`.

Step 4: Extract code for the 'Commit Version Update' stage\
Create a new file called `commitAndPushVersionUpdate.groovy` in the `vars` folder with the following content:
```groovy
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
```

Replace the content of the `script` block in the 'Commit Version Update' stage with `commitAndPushVersionUpdate('github.com/elemasamuel/devops-bootcamp-node-project.git', 'GitHub', 'shared-library')`. ('shared-library' is the name of the branch used for the modifications of the Jenkinsfile calling shared library functions.)

</details>

******
