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
        
        // Application settings
        THINGSBOARD_HOME = '/home/nethmi/Projects/thingsboard'
        THINGSBOARD_PORT = '8080'
        THINGSBOARD_PID_FILE = '/tmp/thingsboard.pid'
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
                            echo "Initial build - will build all modules"
                            changedModules = ['full-build'] // Special marker for full build
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
                        env.CHANGED_MODULES = "full-build"
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
        
        stage('Check Running Application') {
            steps {
                echo 'Checking if ThingsBoard is currently running...'
                script {
                    def isRunning = sh(
                        script: '''
                            # Check if application is running on port 8080
                            if curl -s -f http://localhost:${THINGSBOARD_PORT} > /dev/null 2>&1; then
                                echo "true"
                            elif [ -f "${THINGSBOARD_PID_FILE}" ] && ps -p $(cat ${THINGSBOARD_PID_FILE}) > /dev/null 2>&1; then
                                echo "true"
                            else
                                echo "false"
                            fi
                        ''',
                        returnStdout: true
                    ).trim()
                    
                    env.APP_RUNNING = isRunning
                    
                    if (isRunning == "true") {
                        echo "✅ ThingsBoard application is currently running"
                        echo "Will perform hot deployment of changed modules"
                    } else {
                        echo "⚠️  ThingsBoard application is not running"
                        echo "Will perform full build and start application"
                        env.CHANGED_MODULES = "full-build"
                    }
                }
            }
        }
        
        stage('Build Changed Modules Only') {
            when {
                expression { 
                    env.CHANGED_MODULES != '' && 
                    env.CHANGED_MODULES != 'full-build' && 
                    env.APP_RUNNING == 'true' 
                }
            }
            steps {
                echo 'Building only changed modules (hot deployment)...'
                timeout(time: 20, unit: 'MINUTES') {
                    sh '''
                        echo "=== Hot Deployment: Building Changed Modules Only ==="
                        
                        # Build changed modules using Maven reactor
                        MODULE_LIST=""
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                MODULE_LIST="$MODULE_LIST -pl $module"
                                echo "Will build module: $module"
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
                            
                            # Show built artifacts
                            echo "=== Built Module Artifacts ==="
                            for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                                if [ -d "$module/target" ]; then
                                    echo "Module: $module"
                                    find "$module/target" -name "*.jar" -not -name "*-tests.jar" -not -name "*-sources.jar" | head -5
                                fi
                            done
                        else
                            echo "❌ No valid modules to build"
                            exit 1
                        fi
                    '''
                }
            }
        }
        
        stage('Install Changed Modules') {
            when {
                expression { 
                    env.CHANGED_MODULES != '' && 
                    env.CHANGED_MODULES != 'full-build' && 
                    env.APP_RUNNING == 'true' 
                }
            }
            steps {
                echo 'Installing changed modules to local repository...'
                timeout(time: 10, unit: 'MINUTES') {
                    sh '''
                        echo "=== Installing Changed Modules to Local Repository ==="
                        
                        # Install changed modules to local Maven repository
                        MODULE_LIST=""
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                MODULE_LIST="$MODULE_LIST -pl $module"
                            fi
                        done
                        
                        if [ -n "$MODULE_LIST" ]; then
                            mvn install \
                                $MODULE_LIST \
                                -DskipTests \
                                -Dmaven.test.skip=true \
                                -Dmaven.javadoc.skip=true \
                                -Dmaven.source.skip=true \
                                -Dcheckstyle.skip=true \
                                -Dspotbugs.skip=true \
                                -Dpmd.skip=true \
                                -Dfindbugs.skip=true \
                                -Denforcer.skip=true \
                                -q
                            
                            echo "✅ Changed modules installed to local repository"
                        fi
                    '''
                }
            }
        }
        
        stage('Hot Deploy to Running Application') {
            when {
                expression { 
                    env.CHANGED_MODULES != '' && 
                    env.CHANGED_MODULES != 'full-build' && 
                    env.APP_RUNNING == 'true' 
                }
            }
            steps {
                echo 'Hot deploying changed modules to running application...'
                script {
                    sh '''
                        echo "=== Hot Deployment Process ==="
                        
                        # Create backup directory
                        BACKUP_DIR="backup/$(date +%Y%m%d_%H%M%S)"
                        mkdir -p "$BACKUP_DIR"
                        
                        # Find running application directory (assuming it's running from target)
                        APP_DIR="./application/target"
                        MAIN_JAR=$(find "$APP_DIR" -name "thingsboard-*.jar" -not -name "*-boot.jar" -not -name "*-tests.jar" | head -1)
                        BOOT_JAR=$(find "$APP_DIR" -name "*-boot.jar" | head -1)
                        
                        if [ -n "$BOOT_JAR" ]; then
                            MAIN_JAR="$BOOT_JAR"
                            echo "Using Spring Boot JAR: $MAIN_JAR"
                        fi
                        
                        if [ -z "$MAIN_JAR" ]; then
                            echo "❌ Main application JAR not found!"
                            exit 1
                        fi
                        
                        echo "Main application JAR: $MAIN_JAR"
                        
                        # For hot deployment, we need to rebuild the main Boot JAR with updated modules
                        echo "Hot deployment: Rebuilding main application with updated modules..."
                        
                        # Since ThingsBoard uses Spring Boot fat JAR, we need to rebuild the application module
                        # to include the updated dependencies
                        if echo "$CHANGED_MODULES" | grep -q "application"; then
                            echo "Application module changed - rebuilding Boot JAR..."
                        else
                            echo "Dependency modules changed - rebuilding application with new dependencies..."
                        fi
                        
                        # Rebuild only the application module with updated dependencies
                        cd application
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
                            -q
                        cd ..
                        
                        # Check if new Boot JAR was created
                        NEW_BOOT_JAR="./application/target/thingsboard-4.2.0-SNAPSHOT-boot.jar"
                        if [ ! -f "$NEW_BOOT_JAR" ]; then
                            echo "❌ Failed to rebuild Boot JAR"
                            exit 1
                        fi
                        
                        # Create restart script for the application
                        cat > restart_app.sh << 'EOF'
#!/bin/bash
echo "Restarting ThingsBoard application with updated modules..."

# Stop current application
if [ -f "${THINGSBOARD_PID_FILE}" ]; then
    OLD_PID=$(cat ${THINGSBOARD_PID_FILE})
    if ps -p $OLD_PID > /dev/null 2>&1; then
        echo "Stopping application (PID: $OLD_PID)..."
        kill $OLD_PID
        sleep 15
        if ps -p $OLD_PID > /dev/null 2>&1; then
            echo "Force killing application..."
            kill -9 $OLD_PID
            sleep 5
        fi
        echo "Application stopped"
    fi
fi

# Start application with updated Boot JAR
MAIN_JAR="./application/target/thingsboard-4.2.0-SNAPSHOT-boot.jar"
if [ -f "$MAIN_JAR" ]; then
    echo "Starting application with updated Boot JAR: $MAIN_JAR"
    echo "Boot JAR size: $(ls -lh $MAIN_JAR | awk '{print $5}')"
    
    nohup java -Xmx2048m \
        -XX:+UseG1GC \
        -Dspring.profiles.active=dev \
        -Dlogging.config=classpath:logback.xml \
        -jar "$MAIN_JAR" \
        > logs/thingsboard-restart-${BUILD_NUMBER}.log 2>&1 &
    
    NEW_PID=$!
    echo $NEW_PID > ${THINGSBOARD_PID_FILE}
    echo "Application restarted with PID: $NEW_PID"
    
    # Wait a bit and verify it's running
    sleep 10
    if ps -p $NEW_PID > /dev/null 2>&1; then
        echo "✅ Application is running successfully"
    else
        echo "❌ Application failed to start"
        echo "=== Restart Logs ==="
        tail -50 logs/thingsboard-restart-${BUILD_NUMBER}.log
        exit 1
    fi
else
    echo "❌ Could not find Boot JAR to restart: $MAIN_JAR"
    exit 1
fi
EOF
                        
                        chmod +x restart_app.sh
                        ./restart_app.sh
                        
                        echo "✅ Hot deployment completed"
                    '''
                }
            }
        }
        
        stage('Build Complete Application') {
            when {
                expression { 
                    env.CHANGED_MODULES == 'full-build' || 
                    env.APP_RUNNING != 'true' 
                }
            }
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
                        
                        # Find main application JAR (updated pattern)
                        MAIN_JAR=$(find ./application/target -name "*-boot.jar" | head -1)
                        if [ -z "$MAIN_JAR" ]; then
                            MAIN_JAR=$(find ./application/target -name "thingsboard-*.jar" -not -name "*-tests.jar" | head -1)
                        fi
                        
                        if [ -n "$MAIN_JAR" ]; then
                            echo "Main application JAR: $MAIN_JAR"
                            ls -lh "$MAIN_JAR"
                        else
                            echo "⚠️  Main application JAR not found in expected location"
                            echo "Available JARs in application/target:"
                            ls -la ./application/target/*.jar 2>/dev/null || echo "No JARs found"
                        fi
                    '''
                }
            }
        }
        
        stage('Start ThingsBoard Application') {
            when {
                expression { 
                    env.CHANGED_MODULES == 'full-build' || 
                    env.APP_RUNNING != 'true' 
                }
            }
            steps {
                echo 'Starting ThingsBoard application...'
                script {
                    sh '''
                        echo "=== Starting ThingsBoard Application ==="
                        
                        # Look for the specific ThingsBoard Boot JAR
                        MAIN_JAR="./application/target/thingsboard-4.2.0-SNAPSHOT-boot.jar"
                        
                        if [ ! -f "$MAIN_JAR" ]; then
                            echo "❌ Main ThingsBoard Boot JAR not found at: $MAIN_JAR"
                            echo "Available files in application/target:"
                            ls -la ./application/target/ 2>/dev/null || echo "Directory not found"
                            exit 1
                        fi
                        
                        echo "Found ThingsBoard Boot JAR: $MAIN_JAR"
                        echo "JAR size: $(ls -lh $MAIN_JAR | awk '{print $5}')"
                        
                        # Create logs directory
                        mkdir -p logs
                        
                        # Stop any existing application
                        if [ -f "${THINGSBOARD_PID_FILE}" ]; then
                            OLD_PID=$(cat ${THINGSBOARD_PID_FILE})
                            if ps -p $OLD_PID > /dev/null 2>&1; then
                                echo "Stopping existing application (PID: $OLD_PID)..."
                                kill $OLD_PID
                                sleep 10
                                if ps -p $OLD_PID > /dev/null 2>&1; then
                                    kill -9 $OLD_PID
                                fi
                            fi
                        fi
                        
                        # Start application
                        echo "Starting ThingsBoard application..."
                        nohup java -Xmx2048m \
                            -XX:+UseG1GC \
                            -Dspring.profiles.active=dev \
                            -Dlogging.config=classpath:logback.xml \
                            -jar "$MAIN_JAR" \
                            > logs/thingsboard-${BUILD_NUMBER}.log 2>&1 &
                        
                        APP_PID=$!
                        echo $APP_PID > ${THINGSBOARD_PID_FILE}
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
                            if curl -s -f http://localhost:${THINGSBOARD_PORT} > /dev/null 2>&1; then
                                echo "✅ ThingsBoard application is running and responding!"
                                break
                            elif curl -s http://localhost:${THINGSBOARD_PORT} 2>/dev/null | grep -q "ThingsBoard\\|login\\|dashboard"; then
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
                        echo "Port: $(netstat -tlnp | grep :${THINGSBOARD_PORT} || echo 'Not found')"
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
                        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${THINGSBOARD_PORT} || echo "000")
                        echo "HTTP Response Code: $HTTP_CODE"
                        
                        if [ "$HTTP_CODE" = "200" ] || [ "$HTTP_CODE" = "302" ]; then
                            echo "✅ HTTP endpoint responding correctly"
                        else
                            echo "⚠️  HTTP endpoint returned: $HTTP_CODE"
                        fi
                        
                        # Test specific ThingsBoard endpoints
                        if curl -s http://localhost:${THINGSBOARD_PORT}/login 2>/dev/null | grep -q "ThingsBoard"; then
                            echo "✅ ThingsBoard login page accessible"
                        fi
                        
                        # Check for errors in logs
                        LOG_FILE="logs/thingsboard-${BUILD_NUMBER}.log"
                        if [ ! -f "$LOG_FILE" ]; then
                            LOG_FILE="logs/thingsboard-restart-${BUILD_NUMBER}.log"
                        fi
                        
                        if [ -f "$LOG_FILE" ]; then
                            ERROR_COUNT=$(grep -c "ERROR\\|Exception" "$LOG_FILE" 2>/dev/null || echo "0")
                            if [ "$ERROR_COUNT" -gt 0 ]; then
                                echo "⚠️  Found $ERROR_COUNT errors in logs"
                                echo "Recent errors:"
                                grep "ERROR\\|Exception" "$LOG_FILE" | tail -5
                            else
                                echo "✅ No critical errors found in logs"
                            fi
                        fi
                        
                        echo "=== Build Summary ==="
                        echo "Changed modules: $CHANGED_MODULES"
                        echo "Deployment type: $([ "$APP_RUNNING" = "true" ] && echo "Hot Deployment" || echo "Full Build")"
                        echo "Application URL: http://localhost:${THINGSBOARD_PORT}"
                        echo "Build completed: $(date)"
                    '''
                }
            }
        }
        
        stage('Archive Results') {
            steps {
                echo 'Archiving build artifacts and logs...'
                script {
                    // Archive main application JAR and RPM
                    sh '''
                        # Archive the main deliverables as specified by supervisor
                        
                        # 1. ThingsBoard Boot JAR
                        BOOT_JAR="./application/target/thingsboard-4.2.0-SNAPSHOT-boot.jar"
                        if [ -f "$BOOT_JAR" ]; then
                            cp "$BOOT_JAR" "thingsboard-${BUILD_NUMBER}-boot.jar"
                            echo "✅ Archived Boot JAR: thingsboard-${BUILD_NUMBER}-boot.jar"
                        else
                            echo "⚠️  Boot JAR not found: $BOOT_JAR"
                        fi
                        
                        # 2. ThingsBoard RPM package
                        RPM_FILE="./application/target/thingsboard.rpm"
                        if [ -f "$RPM_FILE" ]; then
                            cp "$RPM_FILE" "thingsboard-${BUILD_NUMBER}.rpm"
                            echo "✅ Archived RPM: thingsboard-${BUILD_NUMBER}.rpm"
                        else
                            echo "⚠️  RPM not found: $RPM_FILE"
                        fi
                        
                        # Also archive the regular JAR for completeness
                        REGULAR_JAR="./application/target/thingsboard-4.2.0-SNAPSHOT.jar"
                        if [ -f "$REGULAR_JAR" ]; then
                            cp "$REGULAR_JAR" "thingsboard-${BUILD_NUMBER}.jar"
                            echo "✅ Archived regular JAR: thingsboard-${BUILD_NUMBER}.jar"
                        fi
                        
                        # Show artifact sizes
                        echo "=== Archived Artifacts ==="
                        ls -lh thingsboard-${BUILD_NUMBER}* 2>/dev/null || echo "No artifacts to show"
                    '''
                    
                    // Archive the main deliverables
                    archiveArtifacts artifacts: 'thingsboard-*-boot.jar,thingsboard-*.rpm,thingsboard-*.jar',
                                   fingerprint: true,
                                   allowEmptyArchive: true
                    
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

=== Deployment Type ===
$([ "$APP_RUNNING" = "true" ] && echo "Hot Deployment (Incremental)" || echo "Full Build")

=== Application Status ===
Boot JAR: ./application/target/thingsboard-4.2.0-SNAPSHOT-boot.jar
RPM Package: ./application/target/thingsboard.rpm
PID File: ${THINGSBOARD_PID_FILE}
URL: http://localhost:${THINGSBOARD_PORT}

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
            echo 'Pipeline completed'
            // Don't stop the application in post-always as we want it to keep running
        }
        
        success {
            echo '✅ ThingsBoard build and deployment succeeded!'
            script {
                try {
                    def modulesBuilt = 0
                    def deploymentType = "Full Build"
                    
                    if (env.CHANGED_MODULES && env.CHANGED_MODULES != '' && env.CHANGED_MODULES != 'full-build') {
                        modulesBuilt = env.CHANGED_MODULES.split(',').size()
                        deploymentType = env.APP_RUNNING == 'true' ? "Hot Deployment" : "Full Build"
                    }
                    
                    currentBuild.description = "✅ ${deploymentType} | ${modulesBuilt} modules | ThingsBoard: http://localhost:${env.THINGSBOARD_PORT}"
                } catch (Exception e) {
                    currentBuild.description = "✅ ThingsBoard build and deployment completed"
                }
            }
        }
        
        failure {
            echo '❌ ThingsBoard build failed!'
            script {
                try {
                    // Stop application if it was started in this build and failed
                    sh '''
                        if [ -f "${THINGSBOARD_PID_FILE}" ]; then
                            APP_PID=$(cat ${THINGSBOARD_PID_FILE})
                            if ps -p $APP_PID > /dev/null 2>&1; then
                                echo "Stopping failed application (PID: $APP_PID)..."
                                kill $APP_PID 2>/dev/null || true
                            fi
                        fi
                    '''
                    
                    def stageName = env.STAGE_NAME ?: 'unknown'
                    currentBuild.description = "❌ Build failed at ${stageName}"
                } catch (Exception e) {
                    currentBuild.description = "❌ Build failed"
                }
            }
        }
    }
}
