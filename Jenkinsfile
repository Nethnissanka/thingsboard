pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx2048m -XX:+UseG1GC'  // Reduced memory allocation
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
            }
        }
        
        stage('Quick Clean') {
            steps {
                echo 'Quick cleanup...'
                sh 'mvn clean -q -T 2C'  // Parallel clean
            }
        }
        
        stage('Fast Package Build') {
            steps {
                echo 'Building packages only - skipping tests and non-essential modules...'
                timeout(time: 15, unit: 'MINUTES') {
                    sh '''
                        # Fast package build with optimizations
                        mvn package \
                            -DskipTests \
                            -Dmaven.test.skip=true \
                            -Dmaven.javadoc.skip=true \
                            -Dmaven.source.skip=true \
                            -Dcheckstyle.skip=true \
                            -Dspotbugs.skip=true \
                            -Dpmd.skip=true \
                            -Dfindbugs.skip=true \
                            -Denforcer.skip=true \
                            -T 2C \
                            -q
                    '''
                }
            }
        }
        
        stage('Archive Essential Packages') {
            steps {
                echo 'Archiving main application packages only...'
                script {
                    // Archive only main application JARs
                    def mainJars = sh(
                        script: "find . -name 'thingsboard*.jar' -o -name 'application*.jar' | grep -v test | head -10",
                        returnStdout: true
                    ).trim()
                    
                    if (mainJars) {
                        echo "Found main packages: ${mainJars}"
                        archiveArtifacts artifacts: '**/target/thingsboard*.jar,**/target/application*.jar', 
                                       excludes: '**/target/*-tests.jar,**/target/*-sources.jar',
                                       allowEmptyArchive: true,
                                       fingerprint: true
                    }
                }
            }
        }
        
        stage('Quick Build Report') {
            steps {
                echo 'Generating quick build summary...'
                sh '''
                    echo "=== Quick ThingsBoard Package Build ===" > build-summary.txt
                    echo "Build: ${BUILD_NUMBER} | Branch: ${BRANCH_NAME}" >> build-summary.txt
                    echo "Date: $(date)" >> build-summary.txt
                    echo "" >> build-summary.txt
                    echo "=== Main Packages Created ===" >> build-summary.txt
                    find . -name "thingsboard*.jar" -o -name "application*.jar" | grep -v test | xargs ls -lh >> build-summary.txt 2>/dev/null || echo "No main packages found" >> build-summary.txt
                '''
                
                archiveArtifacts artifacts: 'build-summary.txt', allowEmptyArchive: true
            }
        }
    }
    
    post {
        always {
            echo 'Quick build pipeline completed!'
            // Minimal cleanup
            sh 'rm -rf target/*/target || true'
        }
        success {
            echo '✅ Quick package build succeeded!'
            script {
                currentBuild.description = "✅ Packages built successfully in ${currentBuild.duration}ms"
            }
        }
        failure {
            echo '❌ Package build failed!'
            script {
                currentBuild.description = "❌ Package build failed at ${env.STAGE_NAME}"
            }
        }
    }
}
