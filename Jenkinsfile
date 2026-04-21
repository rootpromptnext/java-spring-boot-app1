pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = "rootpromptnext"
        IMAGE_NAME = "java-springboot-app1"

        BUILD_TAG = "${BUILD_NUMBER}"
        LATEST_TAG = "latest"

        FULL_IMAGE = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}"

        SNYK_TOKEN = credentials('snyk-token')
        DOCKERHUB_CREDS = credentials('dockerhub-creds')
    }

    stages {

        stage('Checkout Repo') {
            steps {
                git url: 'https://github.com/rootpromptnext/java-spring-boot-app1', branch: 'main'
            }
        }

        stage('Build Maven App (Docker)') {
            steps {
                sh '''
                docker run --rm \
                  -v $PWD:/app \
                  -w /app \
                  maven:3.9.9-eclipse-temurin-17 \
                  mvn clean package -DskipTests
                '''
            }
        }

        stage('SAST - SonarQube (Docker)') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                    docker run --rm \
                      -v $PWD:/usr/src \
                      sonarsource/sonar-scanner-cli \
                      -Dsonar.projectKey=my-app \
                      -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('SCA - Snyk (Docker)') {
            steps {
                sh '''
                docker run --rm \
                  -e SNYK_TOKEN=$SNYK_TOKEN \
                  -v $PWD:/app \
                  -w /app \
                  snyk/snyk:docker \
                  snyk test || true
                '''
            }
        }

        stage('Secrets Scan - TruffleHog (Docker)') {
            steps {
                sh '''
                docker run --rm \
                  -v $PWD:/repo \
                  ghcr.io/trufflesecurity/trufflehog:latest \
                  filesystem /repo || true
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $FULL_IMAGE:$BUILD_TAG .
                docker tag $FULL_IMAGE:$BUILD_TAG $FULL_IMAGE:$LATEST_TAG
                '''
            }
        }

        stage('Trivy Scan (Docker)') {
            steps {
                sh '''
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy:latest \
                  image $FULL_IMAGE:$BUILD_TAG || true
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                sh '''
                echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                docker push $FULL_IMAGE:$BUILD_TAG
                docker push $FULL_IMAGE:$LATEST_TAG
                '''
            }
        }

        stage('Update deployment.yaml') {
            steps {
                sh '''
                sed -i 's|image:.*|image: '$FULL_IMAGE':'$BUILD_TAG'|g' k8s/deployment.yaml
                '''
            }
        }

        stage('Deploy to MicroK8s') {
            steps {
                sh '''
                microk8s kubectl apply -f k8s/deployment.yaml
                microk8s kubectl apply -f k8s/service.yaml
                '''
            }
        }

        stage('DAST - OWASP ZAP (Docker)') {
            steps {
                sh '''
                docker run --rm \
                  -t owasp/zap2docker-stable \
                  zap-baseline.py \
                  -t http://example.com || true
                '''
            }
        }
    }

    post {
        failure {
            emailext(
                subject: "Pipeline Failed: ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
                Build failed.

                Job: ${JOB_NAME}
                Build: ${BUILD_NUMBER}
                URL: ${BUILD_URL}
                """,
                to: "prayag.rhce@gmail.com"
            )
        }
    }
}
