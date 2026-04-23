pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = "rootpromptnext"
        IMAGE_NAME = "java-springboot-app1"

        BUILD_TAG = "${BUILD_NUMBER}"
        LATEST_TAG = "latest"

        FULL_IMAGE = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}"
    }

    stages {

        // =========================
        // Checkout
        // =========================
        stage('Checkout Repo') {
            steps {
                git url: 'https://github.com/rootpromptnext/java-spring-boot-app1', branch: 'main'
            }
        }

        // =========================
        // Build (FIXED permissions)
        // =========================
        stage('Build Maven App (Docker)') {
            steps {
                sh '''
                docker run --rm \
                  -u $(id -u):$(id -g) \
                  -v $PWD:/app \
                  -w /app \
                  maven:3.9.9-eclipse-temurin-17 \
                  mvn clean package -DskipTests
                '''
            }
        }

        // =========================
        // SonarCloud Scan (FIXED)
        // =========================
        stage('SAST - SonarCloud (Docker)') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'TOKEN')]) {
                    sh '''
                    docker run --rm \
                      -v $PWD:/usr/src \
                      -w /usr/src \
                      sonarsource/sonar-scanner-cli \
                      -Dsonar.projectKey=rootpromptnext_java-spring-boot-app1 \
                      -Dsonar.organization=rootpromptnext \
                      -Dsonar.host.url=https://sonarcloud.io \
                      -Dsonar.login=$TOKEN \
                      -Dsonar.sources=src \
                      -Dsonar.java.binaries=target \
                      -Dsonar.projectBaseDir=/usr/src
                    '''
                }
            }
        }

        // =========================
        // Quality Gate (IMPORTANT)
        // =========================
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // =========================
        // Snyk (FIXED)
        // =========================
        stage('SCA - Snyk (Docker)') {
            steps {
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    sh '''
                    docker run --rm \
                      -e SNYK_TOKEN=$SNYK_TOKEN \
                      -v $PWD:/app \
                      -w /app \
                      snyk/snyk:latest \
                      snyk test --file=pom.xml --skip-unresolved --json > snyk-report.json || true

                    docker run --rm \
                      -e SNYK_TOKEN=$SNYK_TOKEN \
                      -v $PWD:/app \
                      -w /app \
                      snyk/snyk:latest \
                      snyk monitor --file=pom.xml --skip-unresolved || true
                    '''
                }
            }
        }

        // =========================
        // Secrets Scan
        // =========================
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

        // =========================
        // Build Docker Image
        // =========================
        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $FULL_IMAGE:$BUILD_TAG .
                docker tag $FULL_IMAGE:$BUILD_TAG $FULL_IMAGE:$LATEST_TAG
                '''
            }
        }

        // =========================
        // Trivy Scan
        // =========================
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

        // =========================
        // Docker Login
        // =========================
        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-token',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    '''
                }
            }
        }

        // =========================
        // Push Image
        // =========================
        stage('Push to Docker Hub') {
            steps {
                sh '''
                docker push $FULL_IMAGE:$BUILD_TAG
                docker push $FULL_IMAGE:$LATEST_TAG
                '''
            }
        }

        // =========================
        // Update Deployment (FIXED sed)
        // =========================
        stage('Update deployment.yaml') {
            steps {
                sh '''
                sed -i "s|image:.*|image: $FULL_IMAGE:$BUILD_TAG|g" deployment.yaml
                '''
            }
        }

        // =========================
        // Deploy to Kubernetes
        // =========================
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    export KUBECONFIG=$KUBECONFIG
                    kubectl apply -f deployment.yaml
                    '''
                }
            }
        }

        // =========================
        // DAST - ZAP
        // =========================
        stage('DAST - OWASP ZAP (Docker)') {
            steps {
                sh '''
                mkdir -p zap-reports
        
                docker run --rm \
                  -v $PWD/zap-reports:/zap/wrk \
                  -t ghcr.io/zaproxy/zaproxy:stable \
                  zap-baseline.py \
                  -t http://learning.rootpromptnext.com \
                  -r zap-report.html || true
                '''
            }
        }
    }

    // =========================
    // POST
    // =========================
    post {
        always {
            archiveArtifacts artifacts: 'zap-reports/*.*, snyk-report.json', allowEmptyArchive: true
        }

        success {
            emailext(
                subject: "Pipeline Success: ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
                <h2>Build Successful</h2>
                <p><b>Job:</b> ${JOB_NAME}</p>
                <p><b>Build:</b> #${BUILD_NUMBER}</p>
                <p><a href="${BUILD_URL}">View Build</a></p>
                """,
                mimeType: 'text/html',
                to: "prayag.rhce@gmail.com"
            )
        }

        failure {
            emailext(
                subject: "Pipeline Failed: ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
                <h2>Build Failed</h2>
                <p><b>Job:</b> ${JOB_NAME}</p>
                <p><b>Build:</b> #${BUILD_NUMBER}</p>
                <p><a href="${BUILD_URL}console">View Console</a></p>
                <p><a href="${BUILD_URL}artifact/">Download Artifacts</a></p>
                """,
                mimeType: 'text/html',
                attachmentsPattern: 'zap-reports/*.html',
                to: "prayag.rhce@gmail.com"
            )
        }
    }
}
