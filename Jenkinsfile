pipeline {

    agent any

    tools {
        maven 'Maven'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    environment {

        AWS_REGION = 'us-east-1'

        APP_NAME = 'java-maven-app'

        ECR_REPO_URL = '846443066184.dkr.ecr.us-east-1.amazonaws.com'

        IMAGE_REPO = "${ECR_REPO_URL}/jenkins-repo"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                url: 'https://github.com/Techy-T/Production-Grade-CI-CD-with-Kubernetes-on-AWS-EKS.git'
            }
        }

        stage('Increment Version') {
            steps {
                script {

                    sh """
                    mvn build-helper:parse-version versions:set \
                    -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                    versions:commit
                    """

                    def pom = readFile('pom.xml')
                    def matcher = pom =~ '<version>(.+?)</version>'

                    env.APP_VERSION = matcher[0][1]
                    env.IMAGE_TAG = "${APP_VERSION}-${BUILD_NUMBER}"

                    echo "Version: ${APP_VERSION}"
                    echo "Image Tag: ${IMAGE_TAG}"
                }
            }
        }

        stage('Build Application') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${IMAGE_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('ECR Login') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
            }
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REPO_URL}
                """
            }
        }

        stage('Push Image') {
            steps {
                sh """
                docker push ${IMAGE_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Configure kubeconfig') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                AWS_PAGER = ''
            }
            steps {
                sh """
                aws eks update-kubeconfig \
                --region ${AWS_REGION} \
                --name demo-cluster
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                export IMAGE_REPO=${IMAGE_REPO}
                export IMAGE_TAG=${IMAGE_TAG}
                export APP_NAME=${APP_NAME}

                envsubst < Kubernetes/deployment.yaml | kubectl apply -f -
                envsubst < Kubernetes/service.yaml | kubectl apply -f -

                kubectl rollout status deployment/${APP_NAME}
                """
            }
        }

        stage('Commit Version Update') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-credentials',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {

                    sh """
                    git config user.email "jenkins@example.com"
                    git config user.name "Jenkins"

                    git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Techy-T/Production-Grade-CI-CD-with-Kubernetes-on-AWS-EKS.git

                    git add pom.xml
                    git commit -m "ci: version bump to ${APP_VERSION}" || true
                    git push origin HEAD:main
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker image prune -af'
            }
        }
    }

    post {
        success {
            echo "SUCCESS: App deployed to demo-cluster"
        }

        failure {
            echo "FAILED: Check logs"
        }

        always {
            echo "Pipeline completed"
        }
    }
}
