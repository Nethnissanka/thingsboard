pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx2048m'
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
            }
        }
        
        stage('Clean') {
            steps {
                echo 'Cleaning previous builds...'
                sh 'mvn clean'
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Compiling source code...'
                sh 'mvn compile -DskipTests'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'mvn test'
            }
            post {
                always {
                    // Publish test results if they exist
                    script {
                        if (fileExists('target/surefire-reports/*.xml')) {
                            publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging application...'
                sh 'mvn package -DskipTests'
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                echo 'Archiving build artifacts...'
                script {
                    // Archive artifacts if they exist
                    if (fileExists('**/target/*.jar')) {
                        archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            cleanWs()
        }
        success {
            echo 'Build succeeded!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
