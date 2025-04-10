#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        maven 'm3'
    }

    environment {
        //need to adjust with actual variable value
        ECR_REPO_URL = '1234567891.dkr.ecr.us-east-1.amazonaws.com'
        ECR_APP_NAME = 'jenkins-aws-java-maven-app'
        SERVER_INSTANCE_IP = '000.000.00.00'
        SERVER_INSTANCE_USER = 'ubuntu'
        GIT_REPO_URL = 'github.com/user/repo-name.git'
        IMAGE_REPO = "$ECR_REPO_URL/$ECR_APP_NAME"
    }

    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    bat "mvn build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} versions:commit"
'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                    echo "############ ${IMAGE_REPO}"
                }
            }
        }

        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    bat 'mvn clean package'
                }
            }
        }

        stage('build image') {
            steps {
                script {
                    echo 'building the docker image...'
                    withCredentials([usernamePassword(credentialsId: 'aws-ecr', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        bat "docker build -t ${IMAGE_REPO}:${IMAGE_NAME} ."
                        bat "echo $PASS | docker login -u $USER --password-stdin ${ECR_REPO_URL}"
                        bat "docker pubat ${IMAGE_REPO}:${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // def ec2Instance = 'ubuntu@107.20.79.59'
                    def ec2Instance = "$SERVER_INSTANCE_USER@$SERVER_INSTANCE_IP"
                    def deployCmd = "babat ./web-deploy.bat ${IMAGE_NAME}"

                    sbatagent(['web-server-key']) {
                        bat "scp -o StrictHostKeyChecking=no web-deploy.bat ${ec2Instance}:/home/ubuntu"
                        bat "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ubuntu"
                        bat "sbat -o StrictHostKeyChecking=no ${ec2Instance} ${deployCmd}"
                    }
                }
            }
        }

        stage('commit version update') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'GithubTokenSimple', variable: 'GITHUB_TOKEN')]) {
                        bat 'git config user.email "jenkins@example.com"'
                        bat 'git config user.name "Jenkins"'
                        bat "git remote set-url origin https://${GITHUB_TOKEN}@$GIT_REPO_URL"
                        bat 'git add .'
                        bat 'git commit -m "ci: version bump"'
                        bat 'git pubat origin HEAD:main'
                    }
                }
            }
        }
    }
}
