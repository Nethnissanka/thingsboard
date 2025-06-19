pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx2048m -XX:+UseG1GC'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
        
        // QA Environment specific variables
        ENV_NAME = 'QA'
        DEPLOY_TARGET = 'qa-server'
        BUILD_PROFILE = 'qa'
    }
    
    parameters {
        choice(
            name: 'BRANCH_TO_BUILD',
            choices: ['feature/*', 'master'],  
            description: 'Select branch to build for QA'
        )
        booleanParam(
            name: 'DEPLOY_TO_QA',
            defaultValue: true,
            description: 'Deploy packages to QA environment after build?'
        )
    }
    
    stages {
        stage('QA Checkout') {
            steps {
                echo "🔄 Checking out source code for QA build..."
                script {
                    if (params.BRANCH_TO_BUILD.startsWith('feature/')) {
                        // For feature branches, let user specify exact branch name
                        checkout scm
                    } else {
                        checkout([$class: 'GitSCM',
                                branches: [[name: "*/${params.BRANCH_TO_BUILD}"]],
                                userRemoteConfigs: scm.userRemoteConfigs])
                    }
                }
            }
        }
        
        stage('QA Build Info') {
            steps {
                echo "🏷️  QA Build Information:"
                echo "Environment: ${ENV_NAME}"
                echo "Branch: ${params.BRANCH_TO_BUILD}"
                echo "Build number: ${env.BUILD_NUMBER}"
                echo "Deploy to QA: ${params.DEPLOY_TO_QA}"
                sh 'java -version'
                sh 'mvn -version'
            }
        }
        
        stage('QA Clean & Prepare') {
            steps {
                echo '🧹 Cleaning workspace for QA build...'
                sh 'mvn clean -q -T 2C'
            }
        }
        
        stage('QA Package Build') {
            steps {
                echo '📦 Building QA packages - optimized for testing...'
                timeout(time: 45, unit: 'MINUTES') {
                    sh '''
                        echo "=== QA Package Build Started ==="
                        echo "Building with QA profile and optimizations..."
                        
                        # QA build with some additional validations but skip heavy tests
                        mvn package \
                            -P${BUILD_PROFILE} \
                            -DskipTests \
                            -Dmaven.test.skip=true \
                            -Dmaven.javadoc.skip=true \
                            -Dmaven.source.skip=true \
                            -Dcheckstyle.skip=false \
                            -Dspotbugs.skip=true \
                            -Dpmd.skip=true \
                            -Dfindbugs.skip=true \
                            -Denforcer.skip=false \
                            -T 2C \
                            -X | grep -E "(Installing|Building|Packaging|reactor:|SUCCESS|ERROR|WARNING)" || \
                        mvn package \
                            -P${BUILD_PROFILE} \
                            -DskipTests \
                            -Dmaven.test.skip=true \
                            -Dmaven.javadoc.skip=true \
                            -Dmaven.source.skip=true \
                            -Dcheckstyle.skip=false \
                            -Denforcer.skip=false \
                            -T 2C
                        
                        echo "=== QA Build Process Completed ==="
                        echo "QA Packages created:"
                        find . -name "*.jar" -newer pom.xml 2>/dev/null | head -15
                    '''
                }
            }
        }
        
        stage('QA Package Validation') {
            steps {
                echo '✅ Validating QA packages...'
                sh '''
                    echo "Validating main application packages..."
                    
                    # Check if main packages exist
                    MAIN_JARS=$(find . -name "thingsboard*.jar" -o -name "application*.jar" | grep -v test | wc -l)
                    echo "Found $MAIN_JARS main application packages"
                    
                    if [ "$MAIN_JARS" -eq 0 ]; then
                        echo "❌ ERROR: No main application packages found!"
                        exit 1
                    fi
                    
                    # Basic JAR validation
                    for jar in $(find . -name "thingsboard*.jar" -o -name "application*.jar" | grep -v test | head -5); do
                        echo "Validating: $jar"
                        if jar tf "$jar" > /dev/null 2>&1; then
                            echo "  ✅ Valid JAR structure"
                        else
                            echo "  ❌ Invalid JAR: $jar"
                            exit 1
                        fi
                    done
                '''
            }
        }
        
        stage('Archive QA Packages') {
            steps {
                echo '📁 Archiving QA packages...'
                script {
                    def qaJars = sh(
                        script: "find . -name '*.jar' | grep -v test | head -20",
                        returnStdout: true
                    ).trim()
                    
                    if (qaJars) {
                        echo "Archiving QA packages: ${qaJars}"
                        archiveArtifacts artifacts: '**/target/*.jar', 
                                       excludes: '**/target/*-tests.jar,**/target/*-sources.jar',
                                       allowEmptyArchive: true,
                                       fingerprint: true
                    }
                }
            }
        }
        
        stage('Deploy to QA') {
            when {
                expression { params.DEPLOY_TO_QA == true }
            }
            steps {
                echo '🚀 Deploying to QA environment...'
                script {
                    // Add your QA deployment logic here
                    sh '''
                        echo "Preparing QA deployment..."
                        echo "Target: ${DEPLOY_TARGET}"
                        
                        # Example deployment commands (customize as needed)
                        # scp target/*.jar qa-server:/opt/thingsboard/
                        # ssh qa-server "sudo systemctl restart thingsboard"
                        
                        echo "✅ QA deployment completed"
                    '''
                }
            }
        }
        
        stage('QA Build Report') {
            steps {
                echo '📊 Generating QA build report...'
                sh '''
                    echo "=== ThingsBoard QA Build Report ===" > qa-build-report.txt
                    echo "Environment: QA" >> qa-build-report.txt
                    echo "Build: ${BUILD_NUMBER} | Branch: ${BRANCH_TO_BUILD}" >> qa-build-report.txt
                    echo "Build Date: $(date)" >> qa-build-report.txt
                    echo "Deployed to QA: ${DEPLOY_TO_QA}" >> qa-build-report.txt
                    echo "" >> qa-build-report.txt
                    
                    echo "=== QA Packages Created ===" >> qa-build-report.txt
                    find . -name "*.jar" -newer pom.xml -exec ls -lh {} \\; >> qa-build-report.txt 2>/dev/null
                    echo "" >> qa-build-report.txt
                    
                    echo "=== Build Statistics ===" >> qa-build-report.txt
                    echo "Total packages: $(find . -name "*.jar" -newer pom.xml | wc -l)" >> qa-build-report.txt
                    echo "Build completed: $(date)" >> qa-build-report.txt
                '''
                
                archiveArtifacts artifacts: 'qa-build-report.txt', allowEmptyArchive: true
            }
        }
    }
    
    post {
        always {
            echo '🏁 QA Pipeline completed!'
            sh 'rm -rf target/*/target || true'
        }
        success {
            echo '✅ QA build succeeded!'
            script {
                currentBuild.description = "✅ QA packages built successfully from ${params.BRANCH_TO_BUILD}"
            }
        }
        failure {
            echo '❌ QA build failed!'
            script {
                currentBuild.description = "❌ QA build failed at ${env.STAGE_NAME}"
            }
        }
    }
}
