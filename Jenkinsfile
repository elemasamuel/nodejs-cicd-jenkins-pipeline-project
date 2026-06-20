#!/usr/bin/env groovy

@Library('jenkins-shared-library')_

pipeline {
    agent any

    parameters {
        booleanParam(name: 'deploy', defaultValue: true, description: 'Deploy the application on the EC2 server.') 
    }

    stages {
        stage('Bump Version') {
            // only execute this stage for the main branch
            when {
                expression {
                    return env.GIT_BRANCH == "origin/main"
                }
            }
            steps {
                script {
                    bumpNpmVersion('app', 'patch')
                }
            }
        }
        stage('Run Tests') {
            // run the tests for every branch
            steps {
                script {
                    runNpmTests('app')
                }
            }
        }
        stage('Build and Push Docker Image') {
            // only execute this stage for the main branch
            when {
                expression {
                    return env.GIT_BRANCH == "origin/main"
                }
            }
            steps {
                buildAndPublishImage("fsiegrist/fesi-repo:devops-bootcamp-node-project-${IMAGE_VERSION}")
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
            // only execute this stage for the main branch
            when {
                expression {
                    return env.GIT_BRANCH == "origin/main"
                }
            }
            steps {
                script {
                    commitAndPushVersionUpdate('github.com/fsiegrist/devops-bootcamp-node-project.git', 'GitHub', 'main')
                }
            }
        }
    }
}