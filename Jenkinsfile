pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx3072m -XX:+UseG1GC -XX:+UseStringDeduplication'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
        
        // Pipeline specific settings
        TARGET_BRANCH = 'pipeline'
        BUILD_PROFILE = 'fast-build'
    }

    triggers {
        githubPush()
    }
    
    stages {
        stage('Validate Branch') {
            steps {
                script {
                    def branchName = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''
                    
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
                    
                    if (branchName.startsWith('origin/')) {
                        branchName = branchName.substring(7)
                    }
                    
                    env.CURRENT_BRANCH = branchName
                    echo "Detected branch: ${branchName}"
                    echo "Target branch: ${TARGET_BRANCH}"
                    
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
                            
                            // Find affected modules based on your structure
                            def moduleSet = [] as Set
                            def knownModules = ['application', 'common', 'dao', 'edqs', 'monitoring', 'netty-mqtt', 
                                             'packaging', 'rest-client', 'rule-engine', 'tools', 'transport', 'ui-ngx', 'msa']
                            
                            changedFiles.each { file ->
                                def parts = file.split('/')
                                if (parts.length > 0) {
                                    def potentialModule = parts[0]
                                    // Check if it's a known module and has pom.xml
                                    if (knownModules.contains(potentialModule)) {
                                        def pomExists = sh(
                                            script: "test -f '${potentialModule}/pom.xml' && echo 'true' || echo 'false'",
                                            returnStdout: true
                                        ).trim() == 'true'
                                        
                                        if (pomExists) {
                                            moduleSet.add(potentialModule)
                                        }
                                    }
                                }
                            }
                            changedModules = moduleSet as List
                        } else {
                            echo "Initial build - will build all modules via root pom"
                            changedModules = ['root'] // Special marker for full build
                        }
                        
                        if (changedModules.isEmpty()) {
                            echo "❌ No changed modules detected. Skipping build to save resources."
                            currentBuild.result = 'NOT_BUILT'
                            return
                        }
                        
                        env.CHANGED_MODULES = changedModules.join(',')
                        echo "Modules that changed: ${env.CHANGED_MODULES}"
                        
                    } catch (Exception e) {
                        echo "Error detecting changed modules: ${e.getMessage()}"
                        env.CHANGED_MODULES = "root"
                    }
                }
            }
        }
        
        stage('Build Info') {
            steps {
                echo "Building ThingsBoard multi-module project"
                echo "Branch: ${env.CURRENT_BRANCH ?: env.BRANCH_NAME ?: 'unknown'}"
                echo "Build number: ${env.BUILD_NUMBER}"
                echo "Changed modules: ${env.CHANGED_MODULES ?: 'detecting...'}"
                sh 'java -version'
                sh 'mvn -version'
                sh 'ls -la'  // Show project structure
            }
        }
        
        stage('Clean Changed Modules') {
            steps {
                echo 'Cleaning only changed modules...'
                sh '''
                    echo "=== Cleaning Changed Modules ==="
                    
                    if [ "$CHANGED_MODULES" = "root" ]; then
                        echo "Full build - cleaning root"
                        mvn clean -q
                    else
                        # Clean only changed modules
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                echo "Cleaning module: $module"
                                cd "$module" && mvn clean -q && cd ..
                            fi
                        done
                    fi
                '''
            }
        }
        
        stage('Build Changed Modules') {
            when {
                expression { env.CHANGED_MODULES != '' && env.CHANGED_MODULES != 'root' }
            }
            steps {
                echo 'Building only changed modules (incremental)...'
                timeout(time: 30, unit: 'MINUTES') {
                    sh '''
                        echo "=== Incremental Build: Changed Modules Only ==="
                        
                        # Build changed modules using Maven reactor
                        MODULE_LIST=""
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                MODULE_LIST="$MODULE_LIST -pl $module"
                            fi
                        done
                        
                        echo "Maven reactor modules: $MODULE_LIST"
                        
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
                            
                            echo "✅ Changed modules built successfully"
                        else
                            echo "❌ No valid modules to build"
                            exit 1
                        fi
                    '''
                }
            }
        }
        
        stage('Build Complete Application') {
            steps {
                echo 'Building complete ThingsBoard application from root pom.xml...'
                timeout(time: 45, unit: 'MINUTES') {
                    sh '''
                        echo "=== Building Complete Application ==="
                        echo "Running: mvn clean package from root directory"
                        
                        # Build the complete application using root pom.xml
                        mvn clean package \
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
                        
                        echo "✅ Complete application built successfully"
                        
                        # Show what was built
                        echo "=== Built Artifacts ==="
                        find . -name "*.jar" -not -path "./.*" -not -name "*-tests.jar" -not -name "*-sources.jar" | head -20
                        
                        # Find main application JAR
                        MAIN_JAR=$(find . -name "*application*.jar" -not -name "*-tests.jar" | head -1)
                        if [ -n "$MAIN_JAR" ]; then
                            echo "Main application JAR: $MAIN_JAR"
                            ls -lh "$MAIN_JAR"
                        fi
                    '''
                }
            }
        }
        
        stage('Run ThingsBoard Application') {
            steps {
                echo 'Starting ThingsBoard application...'
                script {
                    sh '''
                        echo "=== Starting ThingsBoard Application ==="
                        
                        # Find the main application JAR
                        MAIN_JAR=$(find . -name "*application*.jar" -not -name "*-tests.jar" | head -1)
                        
                        if [ -z "$MAIN_JAR" ]; then
                            echo "❌ Main application JAR not found!"
                            echo "Available JARs:"
                            find . -name "*.jar" | grep -v test | head -10
                            exit 1
                        fi
                        
                        echo "Found main JAR: $MAIN_JAR"
                        
                        # Create logs directory
                        mkdir -p logs
                        
                        # Start application (you may need to adjust these settings)
                        echo "Starting ThingsBoard application..."
                        nohup java -Xmx2048m \
                            -XX:+UseG1GC \
                            -Dspring.profiles.active=dev \
                            -Dlogging.config=classpath:logback.xml \
                            -jar "$MAIN_JAR" \
                            > logs/thingsboard-${BUILD_NUMBER}.log 2>&1 &
                        
                        APP_PID=$!
                        echo $APP_PID > app.pid
                        echo "Application started with PID: $APP_PID"
                        
                        # Wait for application to start
                        echo "Waiting for application to start..."
                        TIMEOUT=180  # 3 minutes timeout
                        COUNTER=0
                        
                        while [ $COUNTER -lt $TIMEOUT ]; do
                            # Check if process is still running
                            if ! ps -p $APP_PID > /dev/null 2>&1; then
                                echo "❌ Application process died!"
                                echo "=== Application Logs ==="
                                tail -50 logs/thingsboard-${BUILD_NUMBER}.log
                                exit 1
                            fi
                            
                            # Check if application is responding
                            if curl -s -f http://localhost:8080 > /dev/null 2>&1; then
                                echo "✅ ThingsBoard application is running and responding!"
                                break
                            elif curl -s http://localhost:8080 2>/dev/null | grep -q "ThingsBoard\\|login\\|dashboard"; then
                                echo "✅ ThingsBoard application is running!"
                                break
                            else
                                echo "⏳ Waiting for application... ($COUNTER/$TIMEOUT seconds)"
                                sleep 10
                                COUNTER=$((COUNTER + 10))
                            fi
                        done
                        
                        if [ $COUNTER -ge $TIMEOUT ]; then
                            echo "❌ Application failed to start within timeout"
                            echo "=== Application Logs ==="
                            tail -100 logs/thingsboard-${BUILD_NUMBER}.log
                            exit 1
                        fi
                        
                        # Show application status
                        echo "=== Application Status ==="
                        echo "PID: $APP_PID"
                        echo "Port: $(netstat -tlnp | grep :8080 || echo 'Not found')"
                        echo "Memory: $(ps -o %mem= -p $APP_PID 2>/dev/null || echo 'Unknown')%"
                        
                        # Show recent logs
                        echo "=== Recent Application Logs ==="
                        tail -20 logs/thingsboard-${BUILD_NUMBER}.log
                    '''
                }
            }
        }
        
        stage('Application Health Check') {
            steps {
                echo 'Testing ThingsBoard application...'
                script {
                    sh '''
                        echo "=== ThingsBoard Health Check ==="
                        
                        # Test HTTP endpoint
                        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 || echo "000")
                        echo "HTTP Response Code: $HTTP_CODE"
                        
                        if [ "$HTTP_CODE" = "200" ] || [ "$HTTP_CODE" = "302" ]; then
                            echo "✅ HTTP endpoint responding correctly"
                        else
                            echo "⚠️  HTTP endpoint returned: $HTTP_CODE"
                        fi
                        
                        # Test specific ThingsBoard endpoints
                        if curl -s http://localhost:8080/login 2>/dev/null | grep -q "ThingsBoard"; then
                            echo "✅ ThingsBoard login page accessible"
                        fi
                        
                        # Check for errors in logs
                        ERROR_COUNT=$(grep -c "ERROR\\|Exception" logs/thingsboard-${BUILD_NUMBER}.log 2>/dev/null || echo "0")
                        if [ "$ERROR_COUNT" -gt 0 ]; then
                            echo "⚠️  Found $ERROR_COUNT errors in logs"
                            echo "Recent errors:"
                            grep "ERROR\\|Exception" logs/thingsboard-${BUILD_NUMBER}.log | tail -5
                        else
                            echo "✅ No critical errors found in logs"
                        fi
                        
                        echo "=== Build Summary ==="
                        echo "Changed modules: $CHANGED_MODULES"
                        echo "Application URL: http://localhost:8080"
                        echo "Build completed: $(date)"
                    '''
                }
            }
        }
        
        stage('Archive Results') {
            steps {
                echo 'Archiving build artifacts and logs...'
                script {
                    // Archive main application JAR
                    sh 'find . -name "*application*.jar" -not -name "*-tests.jar" -exec cp {} thingsboard-app-${BUILD_NUMBER}.jar \\;'
                    
                    archiveArtifacts artifacts: 'thingsboard-app-*.jar',
                                   fingerprint: true,
                                   allowEmptyArchive: false
                    
                    // Archive application logs
                    archiveArtifacts artifacts: 'logs/thingsboard-*.log',
                                   allowEmptyArchive: true
                    
                    // Create build report
                    sh '''
                        cat > build-report-${BUILD_NUMBER}.txt << EOF
=== ThingsBoard Build Report ===
Build: ${BUILD_NUMBER}
Branch: ${BRANCH_NAME}
Commit: ${CURRENT_COMMIT}
Date: $(date)

=== Changed Modules ===
${CHANGED_MODULES}

=== Application Status ===
Main JAR: $(find . -name "*application*.jar" -not -name "*-tests.jar" | head -1)
PID: $(cat app.pid 2>/dev/null || echo "Not running")
URL: http://localhost:8080

=== Build Performance ===
Total modules: $(find . -name pom.xml | wc -l)
Changed modules: $(echo ${CHANGED_MODULES} | tr ',' '\\n' | wc -l)
EOF
                    '''
                    
                    archiveArtifacts artifacts: 'build-report-*.txt',
                                   allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up...'
            script {
                try {
                    sh '''
                        # Stop application if running
                        if [ -f "app.pid" ]; then
                            APP_PID=$(cat app.pid)
                            if ps -p $APP_PID > /dev/null 2>&1; then
                                echo "Stopping ThingsBoard application (PID: $APP_PID)..."
                                kill $APP_PID
                                sleep 10
                                # Force kill if still running
                                if ps -p $APP_PID > /dev/null 2>&1; then
                                    kill -9 $APP_PID
                                fi
                                echo "Application stopped"
                            fi
                        fi
                        
                        # Clean up build artifacts but keep logs
                        find . -name "target" -type d -exec rm -rf {} + 2>/dev/null || true
                    '''
                } catch (Exception e) {
                    echo "Cleanup warning: ${e.getMessage()}"
                }
            }
        }
        
        success {
            echo '✅ ThingsBoard build and deployment succeeded!'
            script {
                try {
                    def modulesBuilt = 0
                    if (env.CHANGED_MODULES && env.CHANGED_MODULES != '') {
                        modulesBuilt = env.CHANGED_MODULES.split(',').size()
                    }
                    currentBuild.description = "✅ Built ${modulesBuilt} modules | ThingsBoard running: http://localhost:8080"
                } catch (Exception e) {
                    currentBuild.description = "✅ ThingsBoard build and deployment completed"
                }
            }
        }
        
        failure {
            echo '❌ ThingsBoard build failed!'
            script {
                try {
                    def branchName = env.BRANCH_NAME ?: 'unknown'
                    def stageName = env.STAGE_NAME ?: 'unknown'
                    currentBuild.description = "❌ Build failed at ${stageName}"
                } catch (Exception e) {
                    currentBuild.description = "❌ Build failed"
                }
            }
        }
    }
}
