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
