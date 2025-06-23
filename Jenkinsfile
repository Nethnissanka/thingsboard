pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx3072m -XX:+UseG1GC -XX:+UseStringDeduplication'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
        
        // Pipeline specific settings
        TARGET_BRANCH = 'pipeline'
        COMBINED_JAR_NAME = 'thingsboard-combined'
        BUILD_PROFILE = 'fast-build'
    }
    
    triggers {
        // Webhook trigger for the pipeline branch
        githubPush()
    }
    
    stages {
        stage('Validate Branch') {
            steps {
                script {
                    echo "Current branch: ${env.BRANCH_NAME}"
                    if (env.BRANCH_NAME != TARGET_BRANCH) {
                        error("Pipeline only runs on '${TARGET_BRANCH}' branch. Current branch: ${env.BRANCH_NAME}")
                    }
                }
            }
        }
        
        stage('Checkout & Setup') {
            steps {
                echo 'Checking out pipeline branch...'
                checkout scm
                
                script {
                    // Store commit info for change detection
                    env.CURRENT_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                    
                    env.PREVIOUS_COMMIT = sh(
                        script: 'git rev-parse HEAD~1 2>/dev/null || echo "initial"',
                        returnStdout: true
                    ).trim()
                }
                
                echo "Current commit: ${env.CURRENT_COMMIT}"
                echo "Previous commit: ${env.PREVIOUS_COMMIT}"
            }
        }
        
        stage('Detect Changed Modules') {
            steps {
                echo 'Analyzing changed modules...'
                script {
                    def changedFiles = []
                    def changedModules = []
                    
                    if (env.PREVIOUS_COMMIT != "initial") {
                        // Get changed files
                        changedFiles = sh(
                            script: "git diff --name-only ${env.PREVIOUS_COMMIT}..${env.CURRENT_COMMIT}",
                            returnStdout: true
                        ).trim().split('\n').findAll { it }
                        
                        echo "Changed files: ${changedFiles.join(', ')}"
                        
                        // Find affected modules
                        def moduleSet = [] as Set
                        changedFiles.each { file ->
                            def parts = file.split('/')
                            if (parts.length > 1 && new File(parts[0] + '/pom.xml').exists()) {
                                moduleSet.add(parts[0])
                            }
                        }
                        changedModules = moduleSet as List
                    } else {
                        echo "Initial build - building all modules"
                        changedModules = sh(
                            script: 'find . -name "pom.xml" -not -path "./target/*" -exec dirname {} \\; | grep -v "^\\.$" | sort',
                            returnStdout: true
                        ).trim().split('\n').findAll { it && it != '.' }
                    }
                    
                    env.CHANGED_MODULES = changedModules.join(',')
                    echo "Modules to build: ${env.CHANGED_MODULES}"
                    
                    // Write changed modules to file for later use
                    writeFile file: 'changed-modules.txt', text: env.CHANGED_MODULES
                }
            }
        }
        
        stage('Build Info') {
            steps {
                echo "Building ThingsBoard pipeline branch"
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Build number: ${env.BUILD_NUMBER}"
                echo "Modules to build: ${env.CHANGED_MODULES}"
                sh 'java -version'
                sh 'mvn -version'
            }
        }
        
        stage('Clean & Prepare') {
            steps {
                echo 'Cleaning previous builds...'
                sh '''
                    # Clean only changed modules to save time
                    if [ -n "$CHANGED_MODULES" ]; then
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ]; then
                                echo "Cleaning module: $module"
                                cd "$module" && mvn clean -q && cd ..
                            fi
                        done
                    fi
                    
                    # Clean combined jar directory
                    rm -rf combined-build
                    mkdir -p combined-build/libs
                '''
            }
        }
        
        stage('Build Changed Modules') {
            when {
                expression { env.CHANGED_MODULES != '' }
            }
            steps {
                echo 'Building only changed modules...'
                timeout(time: 45, unit: 'MINUTES') {
                    sh '''
                        echo "=== Building Changed Modules ==="
                        
                        # Create module list for Maven reactor
                        MODULE_LIST=""
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                MODULE_LIST="$MODULE_LIST -pl $module"
                            fi
                        done
                        
                        echo "Maven module list: $MODULE_LIST"
                        
                        if [ -n "$MODULE_LIST" ]; then
                            # Build changed modules with dependencies
                            mvn compile package \
                                $MODULE_LIST \
                                -am \
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
                        else
                            echo "No valid modules to build"
                            exit 1
                        fi
                        
                        echo "=== Module Build Completed ==="
                    '''
                }
            }
        }
        
        stage('Collect & Combine JARs') {
            steps {
                echo 'Collecting built JARs and creating combined package...'
                script {
                    sh '''
                        echo "=== Collecting JAR files ==="
                        
                        # Find all newly built JAR files from changed modules
                        JAR_COUNT=0
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module/target" ]; then
                                echo "Collecting JARs from module: $module"
                                find "$module/target" -name "*.jar" -not -name "*-tests.jar" -not -name "*-sources.jar" -not -name "*-javadoc.jar" | while read jar; do
                                    if [ -f "$jar" ]; then
                                        cp "$jar" combined-build/libs/
                                        echo "Copied: $(basename $jar)"
                                        JAR_COUNT=$((JAR_COUNT + 1))
                                    fi
                                done
                            fi
                        done
                        
                        echo "Total JARs collected: $(ls combined-build/libs/ | wc -l)"
                        ls -la combined-build/libs/
                        
                        # Create combined JAR with all dependencies
                        echo "=== Creating Combined JAR ==="
                        cd combined-build
                        
                        # Create manifest
                        cat > MANIFEST.MF << EOF
Manifest-Version: 1.0
Main-Class: org.thingsboard.server.ThingsboardServerApplication
Implementation-Title: ThingsBoard Combined
Implementation-Version: ${BUILD_NUMBER}
Built-By: Jenkins Pipeline
Build-Branch: ${BRANCH_NAME}
Build-Time: $(date)
EOF
                        
                        # Extract all JARs and combine
                        mkdir -p extracted
                        for jar in libs/*.jar; do
                            if [ -f "$jar" ]; then
                                echo "Extracting: $(basename $jar)"
                                cd extracted
                                jar -xf "../$jar"
                                cd ..
                            fi
                        done
                        
                        # Remove conflicting files
                        find extracted -name "META-INF/*.SF" -delete 2>/dev/null || true
                        find extracted -name "META-INF/*.DSA" -delete 2>/dev/null || true
                        find extracted -name "META-INF/*.RSA" -delete 2>/dev/null || true
                        
                        # Create final combined JAR
                        cd extracted
                        jar -cfm "../${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar" ../MANIFEST.MF *
                        cd ..
                        
                        echo "Combined JAR created: ${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar"
                        ls -lh "${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar"
                        
                        # Verify JAR
                        jar -tf "${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar" | head -10
                    '''
                }
            }
        }
        
        stage('Test Combined JAR') {
            steps {
                echo 'Testing combined JAR integrity...'
                sh '''
                    cd combined-build
                    
                    # Test JAR file integrity
                    echo "Testing JAR integrity..."
                    jar -tf "${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar" > /dev/null
                    
                    if [ $? -eq 0 ]; then
                        echo "✅ JAR integrity test passed"
                    else
                        echo "❌ JAR integrity test failed"
                        exit 1
                    fi
                    
                    # Check main class exists
                    if jar -tf "${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar" | grep -q "org/thingsboard/server/ThingsboardServerApplication.class"; then
                        echo "✅ Main class found in combined JAR"
                    else
                        echo "⚠️  Main class not found - JAR may not be executable"
                    fi
                    
                    # Show JAR details
                    echo "=== Combined JAR Details ==="
                    echo "Size: $(ls -lh ${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar | awk '{print $5}')"
                    echo "Classes: $(jar -tf ${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar | grep '\\.class$' | wc -l)"
                    echo "Resources: $(jar -tf ${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar | grep -v '\\.class$' | wc -l)"
                '''
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                echo 'Archiving combined JAR and build reports...'
                script {
                    // Archive the combined JAR
                    archiveArtifacts artifacts: "combined-build/${env.COMBINED_JAR_NAME}-${env.BUILD_NUMBER}.jar",
                                   fingerprint: true,
                                   allowEmptyArchive: false
                    
                    // Archive individual module JARs
                    archiveArtifacts artifacts: 'combined-build/libs/*.jar',
                                   fingerprint: true,
                                   allowEmptyArchive: true
                    
                    // Create build report
                    sh '''
                        echo "=== ThingsBoard Pipeline Build Report ===" > pipeline-build-report.txt
                        echo "Build: ${BUILD_NUMBER} | Branch: ${BRANCH_NAME}" >> pipeline-build-report.txt
                        echo "Commit: ${CURRENT_COMMIT}" >> pipeline-build-report.txt
                        echo "Date: $(date)" >> pipeline-build-report.txt
                        echo "" >> pipeline-build-report.txt
                        
                        echo "=== Changed Modules ===" >> pipeline-build-report.txt
                        echo "${CHANGED_MODULES}" | tr ',' '\\n' >> pipeline-build-report.txt
                        echo "" >> pipeline-build-report.txt
                        
                        echo "=== Built Artifacts ===" >> pipeline-build-report.txt
                        ls -la combined-build/libs/ >> pipeline-build-report.txt
                        echo "" >> pipeline-build-report.txt
                        
                        echo "=== Combined JAR ===" >> pipeline-build-report.txt
                        ls -lh combined-build/${COMBINED_JAR_NAME}-${BUILD_NUMBER}.jar >> pipeline-build-report.txt
                        echo "" >> pipeline-build-report.txt
                        
                        echo "=== Build Performance ===" >> pipeline-build-report.txt
                        echo "Total modules in project: $(find . -name pom.xml -not -path './target/*' | wc -l)" >> pipeline-build-report.txt
                        echo "Modules built: $(echo ${CHANGED_MODULES} | tr ',' '\\n' | wc -l)" >> pipeline-build-report.txt
                        echo "Build completed at: $(date)" >> pipeline-build-report.txt
                    '''
                    
                    archiveArtifacts artifacts: 'pipeline-build-report.txt, changed-modules.txt',
                                   allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed - cleaning up...'
            sh '''
                # Clean up temporary files but keep combined-build for artifacts
                rm -rf */target/classes 2>/dev/null || true
                rm -f changed-modules.txt 2>/dev/null || true
            '''
        }
        
        success {
            echo '✅ ThingsBoard pipeline build succeeded!'
            script {
                def modulesBuilt = env.CHANGED_MODULES.split(',').size()
                currentBuild.description = "✅ Built ${modulesBuilt} modules | Combined JAR: ${env.COMBINED_JAR_NAME}-${env.BUILD_NUMBER}.jar"
                
                // Webhook notification (optional)
                // You can add webhook notifications here if needed
            }
        }
        
        failure {
            echo '❌ ThingsBoard pipeline build failed!'
            script {
                currentBuild.description = "❌ Build failed at ${env.STAGE_NAME} | Modules: ${env.CHANGED_MODULES}"
            }
            
            // Archive failure logs
            sh '''
                echo "=== Build Failure Report ===" > failure-report.txt
                echo "Failed stage: ${STAGE_NAME}" >> failure-report.txt
                echo "Build: ${BUILD_NUMBER} | Branch: ${BRANCH_NAME}" >> failure-report.txt
                echo "Modules attempted: ${CHANGED_MODULES}" >> failure-report.txt
                echo "Failure time: $(date)" >> failure-report.txt
            '''
            
            archiveArtifacts artifacts: 'failure-report.txt', allowEmptyArchive: true
        }
        
        unstable {
            echo '⚠️  ThingsBoard pipeline build unstable'
            script {
                currentBuild.description = "⚠️  Build unstable | Combined JAR may have issues"
            }
        }
    }
}
