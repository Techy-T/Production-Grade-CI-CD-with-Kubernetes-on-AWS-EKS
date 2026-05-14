
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

        ECR_REPO = 'java-maven-app'

        ECR_REPO_URL = '846443066184.dkr.ecr.us-east-1.amazonaws.com'

        IMAGE_REPO = "${ECR_REPO_URL}/${ECR_REPO}"

        APP_NAME = 'java-maven-app'

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

                    echo 'Incrementing application version...'

                    sh """
                    mvn build-helper:parse-version versions:set \
                    -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                    versions:commit
                    """

                    def pom = readFile('pom.xml')

                    def matcher = pom =~ '<version>(.+?)</version>'

                    env.APP_VERSION = matcher[0][1]

                    env.IMAGE_TAG = "${APP_VERSION}-${BUILD_NUMBER}"

                    echo "Application Version: ${APP_VERSION}"

                    echo "Docker Image Tag: ${IMAGE_TAG}"
                }
            }
        }

        stage('Build Application') {

            steps {

                echo 'Building Maven application...'

                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {

            steps {

                echo 'Building Docker image...'

                sh """
                docker build \
                -t ${IMAGE_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Authenticate to Amazon ECR') {

            environment {
                AWS_ACCESS_KEY_ID     = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
            }

            steps {

                echo 'Authenticating to Amazon ECR...'

                sh """
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login \
                --username AWS \
                --password-stdin ${ECR_REPO_URL}
                """
            }
        }

        stage('Push Docker Image') {

            steps {

                echo 'Pushing Docker image to ECR...'

                sh """
                docker push ${IMAGE_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to EKS') {

            environment {
                AWS_ACCESS_KEY_ID     = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
            }

            steps {

                echo 'Deploying application to EKS...'

                sh """
                export IMAGE_REPO=${IMAGE_REPO}
                export IMAGE_TAG=${IMAGE_TAG}
                export APP_NAME=${APP_NAME}

                envsubst < kubernetes/deployment.yaml | kubectl apply -n ${NAMESPACE} -f -

                envsubst < kubernetes/service.yaml | kubectl apply -n ${NAMESPACE} -f -
                """

                sh """
                kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE}
                """
            }
        }

        stage('Commit Version Update') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-credentials',
                        passwordVariable: 'GIT_PASSWORD',
                        usernameVariable: 'GIT_USERNAME'
                    )
                ]) {

                    echo 'Committing updated version to GitHub...'

                    sh """
                    git config user.email "jenkins@example.com"

                    git config user.name "Jenkins"

                    git remote set-url origin \
                    https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Techy-T/Production-Grade-CI-CD-with-Kubernetes-on-AWS-EKS.git

                    git add pom.xml

                    git commit -m "ci: version bump to ${APP_VERSION}" || true

                    git push origin HEAD:main
                    """
                }
            }
        }

        stage('Cleanup Docker Images') {

            steps {

                echo 'Cleaning unused Docker images...'

                sh """
                docker image prune -af
                """
            }
        }
    }

    post {

        always {

            echo 'Pipeline execution completed.'
        }

        success {

            echo 'Application deployed successfully.'
        }

        failure {

            echo 'Pipeline failed.'
        }
    }
}