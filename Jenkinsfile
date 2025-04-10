#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        maven 'm3'
    }

    environment {
        ECR_REPO_URL = '1234567891.dkr.ecr.us-east-1.amazonaws.com'
        ECR_APP_NAME = 'jenkins-aws-java-maven-app'
        SERVER_INSTANCE_IP = '000.000.00.00'
        SERVER_INSTANCE_USER = 'ubuntu'
        GIT_REPO_URL = 'github.com/user/repo-name.git'
        IMAGE_REPO = "${ECR_REPO_URL}/${ECR_APP_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Increment Version') {
            steps {
                script {
                    echo 'Incrementing app version...'
                    bat '''
                        call mvn build-helper:parse-version versions:set -DnewVersion=%parsedVersion.majorVersion%.%parsedVersion.minorVersion%.%parsedVersion.nextIncrementalVersion% versions:commit
                    '''
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    if (!matcher) {
                        error 'Version not found in pom.xml!'
                    }
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "${version}-${BUILD_NUMBER}"
                    echo "🔹 Image Tag: ${env.IMAGE_REPO}:${env.IMAGE_NAME}"
                }
            }
        }

        stage('Build App') {
            steps {
                script {
                    echo 'Building the application...'
                    bat 'call mvn clean package -DskipTests'
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo 'Building and pushing Docker image...'
                    withCredentials([usernamePassword(credentialsId: 'aws-ecr', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        bat """
                            docker build -t ${env.IMAGE_REPO}:${env.IMAGE_NAME} .
                            echo %PASS% | docker login -u %USER% --password-stdin ${env.ECR_REPO_URL}
                            docker push ${env.IMAGE_REPO}:${env.IMAGE_NAME}
                            docker logout ${env.ECR_REPO_URL}
                        """
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    def ec2Instance = "${SERVER_INSTANCE_USER}@${SERVER_INSTANCE_IP}"
                    def deployCmd = "bash ./web-deploy.bat ${env.IMAGE_NAME}"

                    sshagent(['web-server-key']) {
                        bat """
                            pscp -batch -i path\\to\\key.ppk web-deploy.bat ${ec2Instance}:/home/ubuntu
                            pscp -batch -i path\\to\\key.ppk docker-compose.yaml ${ec2Instance}:/home/ubuntu
                            plink -batch -i path\\to\\key.ppk ${ec2Instance} "${deployCmd}"
                        """
                        // NOTE: `pscp` and `plink` are PuTTY tools. Use if OpenSSH is not available.
                    }
                }
            }
        }

        stage('Commit Version Update') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'GithubTokenSimple', variable: 'GITHUB_TOKEN')]) {
                        bat """
                            git config user.email "jenkins@example.com"
                            git config user.name "Jenkins"
                            git remote set-url origin https://${GITHUB_TOKEN}@${env.GIT_REPO_URL}
                            git add .
                            git commit -m "ci: version bump"
                            git push origin HEAD:main
                        """
                    }
                }
            }
        }
    }
}
