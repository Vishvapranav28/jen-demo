pipeline {
    agent any

    tools {
        maven 'Maven-3.9.11'   // Make sure this matches your Maven installation name in Jenkins
        jdk 'JDK-17'           // Make sure this matches your JDK installation name in Jenkins
    }

    environment {
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building the application...'
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging the application...'
                sh 'mvn package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                    echo 'JAR file created successfully!'
                }
            }
        }

        stage('Code Quality Check') {
            steps {
                echo 'Running code quality checks...'
                sh 'mvn verify -DskipTests'
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo 'Deploying to staging environment...'
                script {
                    try {
                        // Stop any existing Java processes
                        sh '''
                            echo "Checking for existing Java processes..."
                            pgrep -f "demo-1.0.0.jar" || echo "No existing Java processes found"

                            echo "Stopping existing Java processes..."
                            pkill -f "demo-1.0.0.jar" || echo "No process to kill"
                            sleep 3
                        '''

                        // Start the application
                        sh '''
                            echo "Starting the Spring Boot application..."
                            ls -l target
                            nohup java -jar target/demo-1.0.0.jar --server.port=8080 > app.log 2>&1 &
                            echo "Application started. Waiting for startup..."
                            sleep 20
                        '''
                    } catch (Exception e) {
                        echo "Deployment step encountered an issue: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                echo 'Performing application health check...'
                script {
                    def maxRetries = 5
                    def retryCount = 0
                    def healthCheckPassed = false

                    while (retryCount < maxRetries && !healthCheckPassed) {
                        try {
                            sh '''
                                echo "Attempting health check..."
                                curl -f http://localhost:8080/health || exit 1
                            '''
                            healthCheckPassed = true
                            echo "✅ Health check passed!"
                        } catch (Exception e) {
                            retryCount++
                            echo "⚠️ Health check failed (attempt ${retryCount}/${maxRetries}). Retrying in 10 seconds..."
                            sleep(10)
                        }
                    }

                    if (!healthCheckPassed) {
                        error "❌ Health check failed after ${maxRetries} attempts"
                    }
                }
            }
        }

        stage('Integration Tests') {
            steps {
                echo 'Running integration tests...'
                script {
                    def endpoints = ['/', '/hello', '/health']
                    for (ep in endpoints) {
                        try {
                            sh """
                                echo "Testing endpoint ${ep}..."
                                curl -f http://localhost:8080${ep}
                            """
                            echo "✅ Response from ${ep}"
                        } catch (Exception e) {
                            echo "❌ Integration test failed for endpoint ${ep}: ${e.getMessage()}"
                            currentBuild.result = 'FAILURE'
                        }
                    }
                }
            }
        }

        stage('Final Verification') {
            steps {
                echo 'Performing final application verification...'
                sh '''
                    echo "Application is running on http://localhost:8080"
                    echo "Available endpoints:"
                    echo "  - http://localhost:8080/"
                    echo "  - http://localhost:8080/hello"
                    echo "  - http://localhost:8080/health"

                    echo "Checking running Java processes:"
                    ps -ef | grep java
                '''
            }
        }
    }

    post {
        always {
            echo '🏁 Pipeline execution completed!'
            script {
                try {
                    sh 'rm -rf .m2 || true'
                    echo 'Build cache cleaned'
                } catch (Exception e) {
                    echo "Cleanup warning: ${e.getMessage()}"
                }
            }
        }
        success {
            echo '✅ Pipeline executed successfully!'
            echo '🚀 Application is deployed and running on http://localhost:8080'
            echo '📝 Check the application logs if needed (app.log)'
        }
        failure {
            echo '❌ Pipeline failed!'
            echo '🔍 Check the console output above for error details'
            script {
                try {
                    sh 'pkill -f "demo-1.0.0.jar" || echo "No Java processes to kill"'
                    echo 'Stopped application due to pipeline failure'
                } catch (Exception e) {
                    echo "Failed to stop application: ${e.getMessage()}"
                }
            }
        }
        unstable {
            echo '⚠️ Pipeline completed with warnings'
        }
        cleanup {
            echo '🧹 Performing final cleanup...'
            // Uncomment below line if you want to stop the app after every run
            // sh 'pkill -f "demo-1.0.0.jar" || echo "Application stopped"'
        }
    }
}
