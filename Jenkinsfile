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
                    echo '🔧 Incrementing app version...'

                    def pom = readMavenPom file: 'pom.xml'
                    def parts = pom.version.tokenize('.')
                    def major = parts[0]
                    def minor = parts[1]
                    def patch = parts[2].toInteger() + 1
                    def newVersion = "${major}.${minor}.${patch}"

                    echo "🆕 New version: ${newVersion}"

                    bat "mvn versions:set -DnewVersion=${newVersion} versions:commit"

                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    if (!matcher) {
                        error '❌ Version not found in pom.xml!'
                    }
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "${version}-${BUILD_NUMBER}"
                    echo "📦 Docker Image Tag: ${env.IMAGE_REPO}:${env.IMAGE_NAME}"
                }
            }
        }

        stage('Build App') {
            steps {
                echo '🏗️ Building the application...'
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo '🐳 Building and pushing Docker image to ECR...'
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
                    def deployCmd = "web-deploy.bat ${env.IMAGE_NAME}"

                    sshagent(['web-server-key']) {
                        echo '🚀 Deploying to EC2...'
                        bat """
                            pscp web-deploy.bat docker-compose.yaml ${ec2Instance}:/home/ubuntu/
                            plink ${ec2Instance} "${deployCmd}"
                        """
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


