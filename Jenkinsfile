pipeline {
    agent any

    tools {
        maven 'Maven-3.9.11'   // Jenkins -> Global Tool Config
        jdk 'JDK-17'           // Jenkins -> Global Tool Config
    }

    environment {
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📥 Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo '🏗️ Building the application...'
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Running unit tests...'
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
                echo '📦 Packaging the application...'
                sh 'mvn package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                    echo '✅ JAR file created successfully!'
                }
            }
        }

        stage('Code Quality Check') {
            steps {
                echo '🔍 Running code quality checks...'
                sh 'mvn verify -DskipTests'
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo '🚀 Deploying to staging environment...'
                script {
                    sh '''
                        echo "Checking for existing Java processes..."
                        pgrep -f "demo-1.0.0.jar" || echo "No existing Java processes found"

                        echo "Stopping existing Java processes..."
                        pkill -f "demo-1.0.0.jar" || echo "No process to kill"
                        sleep 3

                        echo "Starting the Spring Boot application..."
                        ls -l target
                        nohup java -jar target/demo-1.0.0.jar --server.port=8080 > app.log 2>&1 &
                        echo "Application started. Waiting for startup..."
                        sleep 20
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                echo '🩺 Performing application health check...'
                script {
                    retry(5) {
                        sh '''
                            echo "Attempting health check..."
                            curl -f http://localhost:8080/health || exit 1
                        '''
                    }
                    echo "✅ Health check passed!"
                }
            }
        }

        stage('Integration Tests') {
            steps {
                echo '🔗 Running integration tests...'
                script {
                    def endpoints = ['/health']   // Only test valid endpoints
                    for (ep in endpoints) {
                        sh """
                            echo "Testing endpoint ${ep}..."
                            curl -f http://localhost:8080${ep}
                        """
                        echo "✅ Response from ${ep}"
                    }
                }
            }
        }

        stage('Final Verification') {
            steps {
                echo '🔎 Performing final application verification...'
                sh '''
                    echo "Application is running on http://localhost:8080"
                    echo "Available endpoints:"
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
                sh 'rm -rf .m2 || true'
                echo '🧹 Build cache cleaned'
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
                sh 'pkill -f "demo-1.0.0.jar" || echo "No Java processes to kill"'
                echo '🛑 Stopped application due to pipeline failure'
            }
        }
        unstable {
            echo '⚠️ Pipeline completed with warnings'
        }
    }
}

