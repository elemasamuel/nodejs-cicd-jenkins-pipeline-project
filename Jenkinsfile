#!/usr/bin/env groovy

@Library('jenkins-shared-library')_

pipeline {
    agent any

    parameters {
        booleanParam(name: 'deploy', defaultValue: true, description: 'Deploy the application on the EC2 server.') 
    }

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
                withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]){
                    sh "docker build -t elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-${IMAGE_VERSION} ."
                    sh "echo ${PASSWORD} | docker login -u ${USERNAME} --password-stdin"
                    sh "docker push elemasamuel/sam-repo:nodejs-cicd-jenkins-pipeline-project-${IMAGE_VERSION}"
                }
            }
        }
        stage('Deploy to EC2') {
            // only execute this stage for the main branch and if the respective flag is set
            when {
                expression {
                    return env.GIT_BRANCH == "origin/main" && params.deploy
                }
            }
            steps {
                script {
                    echo 'deploying Docker image to EC2 server...'
                    
                    def dockerComposeCmd = "IMAGE_TAG=${IMAGE_VERSION} docker-compose up -d"
                    def ec2Instance = "ec2-user@3.122.205.189"

                    sshagent(['ec2-server-key']) {
                        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
                        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${dockerComposeCmd}"
                    }
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
