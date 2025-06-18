pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx4096m -XX:+UseG1GC'
        // Use system tools - Rocky Linux paths
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Build Info') {
            steps {
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Build number: ${env.BUILD_NUMBER}"
                sh 'java -version'
                sh 'mvn -version'
                sh 'echo "JAVA_HOME: $JAVA_HOME"'
                sh 'echo "PATH: $PATH"'
                sh 'free -h'  // Check available memory
            }
        }
        
        stage('Clean') {
            steps {
                echo 'Cleaning previous builds...'
                sh 'mvn clean -q'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installing dependencies and building modules in correct order...'
                timeout(time: 30, unit: 'MINUTES') {
                    sh '''
                        # Build in phases to handle complex dependencies
                        echo "Phase 1: Building core modules..."
                        mvn install -DskipTests -pl :common -pl :dao -pl :tools -am -q
                        
                        echo "Phase 2: Building UI module..."
                        mvn install -DskipTests -pl :ui-ngx -am -q
                        
                        echo "Phase 3: Building remaining modules..."
                        mvn install -DskipTests -q
                    '''
                }
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Verifying compilation...'
                sh 'mvn compile -q'
            }
        }
        
        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                timeout(time: 45, unit: 'MINUTES') {
                    sh '''
                        # Run tests on core modules only to save time
                        mvn test -pl :common,:dao,:tools -q
                    '''
                }
            }
            post {
                always {
                    script {
                        // Collect test results from all modules
                        def testReports = sh(
                            script: "find . -name 'surefire-reports' -type d",
                            returnStdout: true
                        ).trim()
                        
                        if (testReports) {
                            publishTestResults testResultsPattern: '**/target/surefire-reports/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging application...'
                timeout(time: 20, unit: 'MINUTES') {
                    sh '''
                        # Package the main application
                        mvn package -DskipTests -pl :application -am -q
                        
                        # Verify the main jar was created
                        find . -name "thingsboard-*.jar" -not -path "*/test/*" | head -5
                    '''
                }
            }
        }
        
        stage('Build Docker Images') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop' 
                    branch 'pipeline'
                }
            }
            steps {
                echo 'Building Docker images for main components...'
                timeout(time: 15, unit: 'MINUTES') {
                    sh '''
                        # Build main application Docker image
                        if [ -f "docker/Dockerfile" ]; then
                            echo "Building ThingsBoard Docker image..."
                            cd docker
                            docker build -t thingsboard:${BUILD_NUMBER} .
                            docker tag thingsboard:${BUILD_NUMBER} thingsboard:latest
                        else
                            echo "No Dockerfile found, skipping Docker build"
                        fi
                    '''
                }
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                echo 'Archiving build artifacts...'
                script {
                    // Archive main application JAR
                    def jarFiles = sh(
                        script: "find . -name '*.jar' -not -path '*/test/*' -not -name '*-tests.jar' -not -name '*-sources.jar'",
                        returnStdout: true
                    ).trim()
                    
                    if (jarFiles) {
                        archiveArtifacts artifacts: '**/target/*.jar', 
                                       excludes: '**/target/*-tests.jar,**/target/*-sources.jar',
                                       allowEmptyArchive: true,
                                       fingerprint: true
                    }
                    
                    // Archive Docker images info if built
                    if (fileExists('docker/Dockerfile')) {
                        sh 'docker images thingsboard:${BUILD_NUMBER} > docker-images.txt || true'
                        archiveArtifacts artifacts: 'docker-images.txt', allowEmptyArchive: true
                    }
                }
            }
        }
        
        stage('Generate Build Report') {
            steps {
                echo 'Generating build report...'
                sh '''
                    echo "=== ThingsBoard Build Report ===" > build-report.txt
                    echo "Build Number: ${BUILD_NUMBER}" >> build-report.txt
                    echo "Branch: ${BRANCH_NAME}" >> build-report.txt
                    echo "Build Date: $(date)" >> build-report.txt
                    echo "Java Version: $(java -version 2>&1 | head -1)" >> build-report.txt
                    echo "Maven Version: $(mvn -version | head -1)" >> build-report.txt
                    echo "" >> build-report.txt
                    echo "=== Artifacts Created ===" >> build-report.txt
                    find . -name "*.jar" -not -path "*/test/*" -exec ls -lh {} \\; >> build-report.txt
                    echo "" >> build-report.txt
                    echo "=== Disk Usage ===" >> build-report.txt
                    du -sh target/ >> build-report.txt 2>/dev/null || echo "No target directory" >> build-report.txt
                '''
                
                archiveArtifacts artifacts: 'build-report.txt', allowEmptyArchive: true
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            script {
                // Clean up Docker images to save space
                sh '''
                    echo "Cleaning up Docker images..."
                    docker image prune -f || true
                '''
            }
            cleanWs()
        }
        success {
            echo '✅ Build succeeded!'
            script {
                currentBuild.description = "✅ Successfully built ThingsBoard ${env.BUILD_NUMBER}"
            }
        }
        failure {
            echo '❌ Build failed!'
            script {
                currentBuild.description = "❌ Build failed at ${env.STAGE_NAME}"
                
                // Capture failure logs
                sh '''
                    echo "=== Build Failure Analysis ===" > failure-analysis.txt
                    echo "Failed Stage: ${STAGE_NAME}" >> failure-analysis.txt
                    echo "Build Number: ${BUILD_NUMBER}" >> failure-analysis.txt
                    echo "Branch: ${BRANCH_NAME}" >> failure-analysis.txt
                    echo "Timestamp: $(date)" >> failure-analysis.txt
                    echo "" >> failure-analysis.txt
                    echo "=== System Resources ===" >> failure-analysis.txt
                    free -h >> failure-analysis.txt
                    df -h >> failure-analysis.txt
                ''' 
                
                archiveArtifacts artifacts: 'failure-analysis.txt', allowEmptyArchive: true
            }
        }
        unstable {
            echo '⚠️ Build unstable - some tests failed'
        }
    }
}
