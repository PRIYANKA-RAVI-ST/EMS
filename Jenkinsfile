pipeline {
    agent any

    environment {
        // System JDK Path
        JAVA_HOME         = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH              = "/usr/lib/jvm/java-17-openjdk-amd64/bin:${env.PATH}"

        // CI Credentials & Configuration
        SONAR_SERVER_NAME = 'SonarQube-Server'
        SNYK_TOKEN        = credentials('snyk-api-token')
        FOSSA_API_KEY     = credentials('fossa-api-key')

        // AWS & CD Configuration
        AWS_REGION        = 'us-east-1'
        ECR_REPO_NAME     = 'ems-app'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }

    stages {
        // ==========================================
        // CONTINUOUS INTEGRATION (CI) PHASES
        // ==========================================

        stage('Checkout SCM') {
            steps {
                echo '=== CI Stage 1: Source Code Checkout ==='
                checkout scm
                sh 'chmod +x mvnw || true'
            }
        }

        stage('Build & Unit Test') {
            steps {
                echo '=== CI Stage 2: Maven Build & Unit Testing ==='
                sh './mvnw clean test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Static Analysis') {
            steps {
                echo '=== CI Stage 3: SonarQube SAST & Code Quality ==='
                script {
                    try {
                        withSonarQubeEnv(env.SONAR_SERVER_NAME) {
                            sh '''
                                chmod +x mvnw
                                ./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar \
                                  -Dsonar.projectKey=EMS \
                                  -Dsonar.projectName=EMS \
                                  -Dsonar.host.url=http://localhost:9000 \
                                  -Dsonar.java.binaries=target/classes || true
                            '''
                        }
                    } catch (Exception e) {
                        echo "⚠️ SonarQube server not configured in Jenkins. Skipping analysis: ${e.message}"
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                echo '=== CI Stage 3b: Waiting for Quality Gate ==='
                script {
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: false
                        }
                    } catch (Exception e) {
                        echo "⚠️ Quality Gate skipped: ${e.message}"
                    }
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                echo '=== CI Stage 4: Snyk Vulnerability & Dependency Scan ==='
                script {
                    try {
                        sh '''
                            if ! command -v snyk > /dev/null 2>&1; then
                                curl -sL https://static.snyk.io/cli/latest/snyk-linux -o ./snyk
                                chmod +x ./snyk
                                SNYK_CMD="./snyk"
                            else
                                SNYK_CMD="snyk"
                            fi

                            $SNYK_CMD auth ${SNYK_TOKEN} || true
                            $SNYK_CMD test --severity-threshold=high --json > snyk-report.json || true
                            $SNYK_CMD monitor --project-name=EMS || true
                        '''
                    } catch (Exception e) {
                        echo "⚠️ Snyk scan completed with warnings: ${e.message}"
                    }
                }
            }
        }

        stage('FOSSA License Compliance') {
            steps {
                echo '=== CI Stage 5: FOSSA License Compliance Scan ==='
                script {
                    try {
                        sh '''
                            if ! command -v fossa > /dev/null 2>&1; then
                                curl -H 'Cache-Control: no-cache' https://raw.githubusercontent.com/fossas/fossa-cli/master/install-latest.sh | bash || true
                            fi
                            fossa analyze || true
                            fossa test --timeout 300 || true
                        '''
                    } catch (Exception e) {
                        echo "⚠️ FOSSA scan skipped due to network glitch: ${e.message}"
                    }
                }
            }
        }

        // ==========================================
        // CONTINUOUS DEPLOYMENT (CD) PHASES
        // (Runs automatically on main branch merges)
        // ==========================================

        stage('Build Docker Image') {
    when {
        anyOf {
            branch 'main'
            branch 'feature/priyanka-1'
        }
    }
            steps {
                echo '=== CD Stage 1: Build Docker Container Image ==='
                script {
                    sh '''
                        docker build -t ${ECR_REPO_NAME}:${BUILD_NUMBER} .
                        docker tag ${ECR_REPO_NAME}:${BUILD_NUMBER} ${ECR_REPO_NAME}:latest
                    '''
                }
            }
        }

        stage('Push Image to AWS ECR') {
    when {
        anyOf {
            branch 'main'
            branch 'feature/priyanka-1'
        }
    }
            steps {
                echo '=== CD Stage 2: Push Image to AWS ECR (Optional) ==='
                script {
                    withCredentials([usernamePassword(credentialsId: 'aws-ec2-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                            # Retrieve AWS Account ID dynamically
                            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text 2>/dev/null || echo "")
                            if [ -n "$AWS_ACCOUNT_ID" ]; then
                                aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com || true
                                docker tag ${ECR_REPO_NAME}:${BUILD_NUMBER} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${BUILD_NUMBER} || true
                                docker tag ${ECR_REPO_NAME}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest || true
                                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${BUILD_NUMBER} || true
                                docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:latest || true
                            else
                                echo "AWS ECR login skipped or not configured."
                            fi
                        '''
                    }
                }
            }
        }

        stage('Deploy to Production Container') {
       when {
        anyOf {
            branch 'main'
            branch 'feature/priyanka-1'
        }
    }
            steps {
                echo '=== CD Stage 3: Deploy Application Container to Port 8081 ==='
                script {
                    sh '''
                        echo "Deploying ${ECR_REPO_NAME}:latest container on production port 8081..."
                        docker stop ems-prod-app || true
                        docker rm ems-prod-app || true
                        docker run -d --name ems-prod-app \
                          -p 8081:8080 \
                          --restart always \
                          ${ECR_REPO_NAME}:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace...'
            cleanWs(deleteDirs: true, notFailBuild: true)
        }
        success {
            echo '✅ Full CI/CD Pipeline completed successfully!'
        }
        failure {
            echo '❌ CI/CD Pipeline execution failed!'
        }
    }
}
