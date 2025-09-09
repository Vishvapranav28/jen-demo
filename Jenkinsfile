pipeline {
    agent any
    
    tools {
        maven 'Maven-3.9.11'  // Make sure this matches your Maven installation name in Jenkins
        jdk 'JDK-17'         // Make sure this matches your JDK installation name in Jenkins
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
                bat 'mvn compile'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                bat 'mvn test'
            }
            post {
                always {
                    // Publish test results
                   // publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging the application...'
                bat 'mvn package -DskipTests'
            }
            post {
                success {
                    // Archive the built artifacts
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                    echo 'JAR file created successfully!'
                }
            }
        }
        
        stage('Code Quality Check') {
            steps {
                echo 'Running code quality checks...'
                bat 'mvn verify -DskipTests'
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                echo 'Deploying to staging environment...'
                script {
                    try {
                        // Stop any existing Java processes
                        bat '''
                            echo "Checking for existing Java processes..."
                            tasklist /fi "imagename eq java.exe" 2>nul
                            echo "Stopping existing Java processes..."
                            taskkill /f /im java.exe 2>nul || echo "No existing Java processes found"
                            timeout /t 3 /nobreak >nul
                        '''
                        
                        // Start the application
                        bat '''
                            echo "Starting the Spring Boot application..."
                            dir target
                            start /b java -jar target\\demo-1.0.0.jar --server.port=8080
                            echo "Application started. Waiting for startup..."
                            timeout /t 20 /nobreak >nul
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
                            bat '''
                                echo "Attempting health check (attempt %RETRY_COUNT%)..."
                                powershell -Command "& {
                                    try {
                                        $response = Invoke-RestMethod -Uri 'http://localhost:8080/health' -TimeoutSec 10
                                        Write-Host 'Health check response:' $response
                                        exit 0
                                    } catch {
                                        Write-Host 'Health check failed:' $_.Exception.Message
                                        exit 1
                                    }
                                }"
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
                    bat """
                        powershell -Command "& {
                            try {
                                \$response = Invoke-RestMethod -Uri 'http://localhost:8080${ep}' -TimeoutSec 10
                                Write-Host '✅ Response from ${ep}: ' \$response
                                exit 0
                            } catch {
                                Write-Host '❌ Failed ${ep}: ' \$_.Exception.Message
                                exit 1
                            }
                        }"
                    """
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
                bat '''
                    echo "Application is running on http://localhost:8080"
                    echo "Available endpoints:"
                    echo "  - http://localhost:8080/"
                    echo "  - http://localhost:8080/hello" 
                    echo "  - http://localhost:8080/health"
                    
                    echo "Checking running Java processes:"
                    tasklist /fi "imagename eq java.exe"
                '''
            }
        }
    }
    
    post {
        always {
            echo '🏁 Pipeline execution completed!'
            // Clean workspace but keep the running application
            script {
                try {
                    // Clean up build artifacts but keep target directory
                    bat 'if exist ".m2" rmdir /s /q .m2'
                    echo 'Build cache cleaned'
                } catch (Exception e) {
                    echo "Cleanup warning: ${e.getMessage()}"
                }
            }
        }
        success {
            echo '✅ Pipeline executed successfully!'
            echo '🚀 Application is deployed and running on http://localhost:8080'
            echo '📝 Check the application logs if needed'
        }
        failure {
            echo '❌ Pipeline failed!'
            echo '🔍 Check the console output above for error details'
            // Stop application on failure
            script {
                try {
                    bat 'taskkill /f /im java.exe 2>nul || echo "No Java processes to kill"'
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
            // Optional: Stop the application after a successful run
            // Uncomment the next line if you want to stop the app after each pipeline run
            // bat 'taskkill /f /im java.exe 2>nul || echo "Application stopped"'
        }
    }
}
