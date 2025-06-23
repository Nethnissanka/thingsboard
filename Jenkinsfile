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
                    // Get branch name from multiple sources
                    def branchName = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''
                    
                    // If still empty, try to get from git
                    if (!branchName || branchName == '') {
                        try {
                            branchName = sh(
                                script: 'git branch --show-current 2>/dev/null || git rev-parse --abbrev-ref HEAD',
                                returnStdout: true
                            ).trim()
                        } catch (Exception e) {
                            echo "Could not determine branch name: ${e.getMessage()}"
                            branchName = 'unknown'
                        }
                    }
                    
                    // Remove origin/ prefix if present
                    if (branchName.startsWith('origin/')) {
                        branchName = branchName.substring(7)
                    }
                    
                    // Set the branch name for use in other stages
                    env.CURRENT_BRANCH = branchName
                    
                    echo "Detected branch: ${branchName}"
                    echo "Target branch: ${TARGET_BRANCH}"
                    
                    // Only enforce branch restriction if we can determine the branch
                    if (branchName != 'unknown' && branchName != TARGET_BRANCH) {
                        echo "⚠️  Warning: Pipeline designed for '${TARGET_BRANCH}' branch, but running on '${branchName}'"
                        echo "Continuing with build..."
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
                    
                    try {
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
                                if (parts.length > 1) {
                                    // Check if directory has pom.xml
                                    def pomPath = "${parts[0]}/pom.xml"
                                    def pomExists = sh(
                                        script: "test -f '${pomPath}' && echo 'true' || echo 'false'",
                                        returnStdout: true
                                    ).trim() == 'true'
                                    
                                    if (pomExists) {
                                        moduleSet.add(parts[0])
                                    }
                                }
                            }
                            changedModules = moduleSet as List
                        } else {
                            echo "Initial build - finding all modules"
                            def allModules = sh(
                                script: 'find . -name "pom.xml" -not -path "./target/*" -exec dirname {} \\; | grep -v "^\\.$" | sed "s|^\\./||" | sort',
                                returnStdout: true
                            ).trim()
                            
                            if (allModules) {
                                changedModules = allModules.split('\n').findAll { it && it != '.' }
                            }
                        }
                        
                        // // If no modules found, try to find some common ThingsBoard modules
                        // if (changedModules.isEmpty()) {
                        //     echo "No changed modules detected, looking for common modules..."
                        //     def commonModules = sh(
                        //         script: '''
                        //             for module in application common dao transport netty-mqtt rule-engine ui-ngx; do
                        //                 if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                        //                     echo "$module"
                        //                 fi
                        //             done
                        //         ''',
                        //         returnStdout: true
                        //     ).trim()
                            
                        //     if (commonModules) {
                        //         changedModules = commonModules.split('\n').findAll { it }
                        //     }
                        // }


                        if (changedModules.isEmpty()) {
    echo "❌ No changed modules detected. Skipping build to save resources."
    currentBuild.result = 'NOT_BUILT'
    return
}
                        
                        env.CHANGED_MODULES = changedModules.join(',')
                        echo "Modules to build: ${env.CHANGED_MODULES}"
                        
                        // Write changed modules to file for later use
                        writeFile file: 'changed-modules.txt', text: env.CHANGED_MODULES
                        
                    } catch (Exception e) {
                        echo "Error detecting changed modules: ${e.getMessage()}"
                        // Fallback: try to build root project
                        env.CHANGED_MODULES = "."
                        writeFile file: 'changed-modules.txt', text: env.CHANGED_MODULES
                    }
                }
            }
        }
        
        stage('Build Info') {
            steps {
                echo "Building ThingsBoard pipeline branch"
                echo "Branch: ${env.CURRENT_BRANCH ?: env.BRANCH_NAME ?: 'unknown'}"
                echo "Build number: ${env.BUILD_NUMBER}"
                echo "Modules to build: ${env.CHANGED_MODULES ?: 'detecting...'}"
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
            script {
                try {
                    sh '''
                        # Clean up temporary files but keep combined-build for artifacts
                        rm -rf */target/classes 2>/dev/null || true
                        rm -f changed-modules.txt 2>/dev/null || true
                    '''
                } catch (Exception e) {
                    echo "Cleanup warning: ${e.getMessage()}"
                }
            }
        }
        
        success {
            echo '✅ ThingsBoard pipeline build succeeded!'
            script {
                try {
                    def modulesBuilt = 0
                    if (env.CHANGED_MODULES && env.CHANGED_MODULES != '') {
                        modulesBuilt = env.CHANGED_MODULES.split(',').size()
                    }
                    currentBuild.description = "✅ Built ${modulesBuilt} modules | Combined JAR: ${env.COMBINED_JAR_NAME ?: 'thingsboard-combined'}-${env.BUILD_NUMBER}.jar"
                } catch (Exception e) {
                    echo "Success description error: ${e.getMessage()}"
                    currentBuild.description = "✅ Build completed successfully"
                }
            }
        }
        
        failure {
            echo '❌ ThingsBoard pipeline build failed!'
            script {
                try {
                    // Safe variable handling
                    def branchName = env.BRANCH_NAME ?: 'unknown'
                    def stageName = env.STAGE_NAME ?: 'unknown'
                    def changedModules = env.CHANGED_MODULES ?: 'none'
                    def buildNumber = env.BUILD_NUMBER ?: 'unknown'
                    
                    currentBuild.description = "❌ Build failed at ${stageName} | Branch: ${branchName}"
                    
                    // Create failure report with safe variables
                    writeFile file: 'failure-report.txt', text: """=== Build Failure Report ===
Failed stage: ${stageName}
Build: ${buildNumber}
Branch: ${branchName}
Modules attempted: ${changedModules}
Failure time: ${new Date()}
Workspace: ${env.WORKSPACE ?: 'unknown'}
Node: ${env.NODE_NAME ?: 'unknown'}
"""
                    
                    archiveArtifacts artifacts: 'failure-report.txt', allowEmptyArchive: true
                    
                } catch (Exception e) {
                    echo "Post-failure processing error: ${e.getMessage()}"
                    currentBuild.description = "❌ Build failed with post-processing errors"
                }
            }
        }
        
        unstable {
            echo '⚠️  ThingsBoard pipeline build unstable'
            script {
                try {
                    currentBuild.description = "⚠️  Build unstable | Combined JAR may have issues"
                } catch (Exception e) {
                    echo "Unstable description error: ${e.getMessage()}"
                }
            }
        }
    }
}
