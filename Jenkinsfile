pipeline {
    agent any
    
    // Webhook trigger configuration
    triggers {
        // Git webhook trigger - automatically triggers on push/PR events
        githubPush()
        
        // Alternative: Generic webhook trigger (uncomment if using GitLab or other Git providers)
        // GenericTrigger(
        //     genericVariables: [
        //         [key: 'ref', value: '$.ref'],
        //         [key: 'repository_name', value: '$.repository.name'],
        //         [key: 'pusher_name', value: '$.pusher.name']
        //     ],
        //     token: 'your-webhook-token-here',
        //     printContributedVariables: true,
        //     printPostContent: true
        // )
    }
    
    environment {
        MAVEN_OPTS = '-Xmx2048m -XX:+UseG1GC'  // Reduced memory allocation
        // Use system tools - Rocky Linux paths
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
        
        // Webhook related variables
        WEBHOOK_TRIGGERED = "${env.GITHUB_WEBHOOK_TRIGGERED ?: 'false'}"
    }
    
    stages {
        stage('Webhook Info & Checkout') {
            steps {
                script {
                    if (env.WEBHOOK_TRIGGERED == 'true') {
                        echo '🔗 Build triggered by webhook'
                        echo "Repository: ${env.GITHUB_REPOSITORY ?: 'N/A'}"
                        echo "Branch: ${env.GITHUB_REF ?: env.BRANCH_NAME}"
                        echo "Commit: ${env.GITHUB_SHA ?: 'N/A'}"
                        echo "Pusher: ${env.GITHUB_ACTOR ?: 'N/A'}"
                    } else {
                        echo '👤 Build triggered manually'
                    }
                }
                
                echo 'Checking out source code...'
                checkout scm
                
                // Display recent commits for context
                sh '''
                    echo "=== Recent Commits ==="
                    git log --oneline -5 || echo "Could not fetch git log"
                    echo "Current branch: $(git branch --show-current 2>/dev/null || echo 'unknown')"
                '''
            }
        }
        
        stage('Build Info') {
            steps {
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Build number: ${env.BUILD_NUMBER}"
                echo "Webhook triggered: ${env.WEBHOOK_TRIGGERED}"
                
                script {
                    if (env.WEBHOOK_TRIGGERED == 'true') {
                        echo "🔗 Webhook build details:"
                        echo "  - Trigger source: ${env.GITHUB_EVENT_NAME ?: 'push'}"
                        echo "  - Repository: ${env.GITHUB_REPOSITORY ?: 'N/A'}"
                        echo "  - Ref: ${env.GITHUB_REF ?: 'N/A'}"
                    }
                }
                
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
                timeout(time: 60, unit: 'MINUTES') {
                    sh '''
                        echo "=== Starting Maven Package Build ==="
                        echo "Installing dependencies and building packages..."
                        
                        # Fast package build with optimizations and verbose output for packages
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
                            -X | grep -E "(Installing|Building|Packaging|reactor:|SUCCESS|ERROR)" || \
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
                            -T 2C
                        
                        echo "=== Build Process Completed ==="
                        echo "Listing all created packages:"
                        find . -name "*.jar" -newer pom.xml 2>/dev/null | head -20
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
        
        stage('Build Report & Package Analysis') {
            steps {
                echo 'Generating detailed build summary...'
                sh '''
                    echo "=== ThingsBoard Package Build Report ===" > build-summary.txt
                    echo "Build: ${BUILD_NUMBER} | Branch: ${BRANCH_NAME}" >> build-summary.txt
                    echo "Date: $(date)" >> build-summary.txt
                    echo "Build Duration: Started at $(date)" >> build-summary.txt
                    echo "" >> build-summary.txt
                    
                    echo "=== Maven Modules Processed ===" >> build-summary.txt
                    find . -name "pom.xml" -exec dirname {} \\; | sort >> build-summary.txt
                    echo "" >> build-summary.txt
                    
                    echo "=== All Packages/JARs Created ===" >> build-summary.txt
                    find . -name "*.jar" -newer pom.xml -exec ls -lh {} \\; >> build-summary.txt 2>/dev/null || echo "No new JAR files found" >> build-summary.txt
                    echo "" >> build-summary.txt
                    
                    echo "=== Main Application Packages ===" >> build-summary.txt
                    find . -name "thingsboard*.jar" -o -name "application*.jar" | grep -v test | xargs ls -lh >> build-summary.txt 2>/dev/null || echo "No main packages found" >> build-summary.txt
                    echo "" >> build-summary.txt
                    
                    echo "=== Package Dependencies Installed ===" >> build-summary.txt
                    find ~/.m2/repository -name "*.jar" -newer pom.xml | wc -l > dependency-count.tmp 2>/dev/null || echo "0" > dependency-count.tmp
                    echo "New dependencies downloaded: $(cat dependency-count.tmp)" >> build-summary.txt
                    rm -f dependency-count.tmp
                    
                    echo "" >> build-summary.txt
                    echo "=== Build Summary ===" >> build-summary.txt
                    echo "Total JAR files created: $(find . -name "*.jar" -newer pom.xml | wc -l)" >> build-summary.txt
                    echo "Build completed at: $(date)" >> build-summary.txt
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
                def triggerInfo = env.WEBHOOK_TRIGGERED == 'true' ? ' (webhook)' : ' (manual)'
                currentBuild.description = "✅ Packages built successfully${triggerInfo} in ${currentBuild.duration}ms"
                
                // Send webhook notification on success (optional)
                if (env.WEBHOOK_TRIGGERED == 'true') {
                    echo '📤 Sending success notification...'
                    // Add your notification logic here
                    // Example: Slack, Teams, or custom webhook notification
                }
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
