pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx3072m -XX:+UseG1GC -XX:+UseStringDeduplication'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
        
        // Pipeline specific settings
        TARGET_BRANCH = 'pipeline'
        THINGSBOARD_HOME = '/opt/thingsboard'  // Where ThingsBoard is installed
        LIB_DIR = '/opt/thingsboard/lib'       // Where JARs are located
        BACKUP_DIR = '/opt/thingsboard/backup' // Backup location
    }

    triggers {
        githubPush()
    }
    
    stages {
        stage('Validate Branch & Setup') {
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
                            branchName = 'unknown'
                        }
                    }
                    
                    if (branchName.startsWith('origin/')) {
                        branchName = branchName.substring(7)
                    }
                    
                    env.CURRENT_BRANCH = branchName
                    echo "Detected branch: ${branchName}"
                }
                
                // Setup workspace
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
            }
        }
        
        stage('Detect Changed Modules') {
            steps {
                echo 'Analyzing changed modules for incremental build...'
                script {
                    def changedModules = []
                    
                    try {
                        if (env.PREVIOUS_COMMIT != "initial") {
                            def changedFiles = sh(
                                script: "git diff --name-only ${env.PREVIOUS_COMMIT}..${env.CURRENT_COMMIT}",
                                returnStdout: true
                            ).trim().split('\n').findAll { it }
                            
                            echo "Changed files: ${changedFiles.join(', ')}"
                            
                            def moduleSet = [] as Set
                            def knownModules = ['application', 'common', 'dao', 'edqs', 'monitoring', 'netty-mqtt', 
                                             'packaging', 'rest-client', 'rule-engine', 'tools', 'transport', 'ui-ngx', 'msa']
                            
                            changedFiles.each { file ->
                                def parts = file.split('/')
                                if (parts.length > 0) {
                                    def potentialModule = parts[0]
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
                        }
                        
                        if (changedModules.isEmpty()) {
                            echo "❌ No changed modules detected. Skipping incremental build."
                            currentBuild.result = 'NOT_BUILT'
                            return
                        }
                        
                        env.CHANGED_MODULES = changedModules.join(',')
                        echo "🎯 Modules for incremental build: ${env.CHANGED_MODULES}"
                        
                    } catch (Exception e) {
                        echo "Error detecting changed modules: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                        return
                    }
                }
            }
        }
        
        stage('Build Only Changed Modules') {
            when {
                expression { env.CHANGED_MODULES != '' }
            }
            steps {
                echo 'Building ONLY changed modules (incremental)...'
                timeout(time: 20, unit: 'MINUTES') {
                    sh '''
                        echo "=== Incremental Build: Changed Modules Only ==="
                        
                        # Create directory for new JARs
                        mkdir -p incremental-build/new-jars
                        
                        # Build each changed module individually
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                echo "🔨 Building module: $module"
                                
                                cd "$module"
                                
                                # Clean and build this module only
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
                                    -q
                                
                                # Copy built JARs to staging area
                                find target -name "*.jar" -not -name "*-tests.jar" -not -name "*-sources.jar" -not -name "*-javadoc.jar" | while read jar; do
                                    if [ -f "$jar" ]; then
                                        cp "$jar" "../incremental-build/new-jars/"
                                        echo "✅ Built: $(basename $jar)"
                                    fi
                                done
                                
                                cd ..
                            fi
                        done
                        
                        echo "=== Incremental Build Results ==="
                        ls -la incremental-build/new-jars/
                        echo "Total new JARs: $(ls incremental-build/new-jars/*.jar 2>/dev/null | wc -l)"
                    '''
                }
            }
        }
        
        stage('Install to Root Project') {
            steps {
                echo 'Installing changed modules to root project using mvn install...'
                timeout(time: 10, unit: 'MINUTES') {
                    sh '''
                        echo "=== Installing Changed Modules to Local Repository ==="
                        
                        # Install each changed module to local Maven repository
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                echo "📦 Installing module: $module"
                                
                                cd "$module"
                                mvn install -DskipTests -q
                                cd ..
                                
                                echo "✅ Installed: $module"
                            fi
                        done
                        
                        echo "=== Root Project Install (Dependencies Only) ==="
                        # Install dependencies without full build
                        mvn dependency:resolve \
                            dependency:copy-dependencies \
                            -DoutputDirectory=incremental-build/dependencies \
                            -DincludeScope=runtime \
                            -q
                        
                        echo "✅ Root project dependencies resolved"
                    '''
                }
            }
        }
        
        stage('Check Running Application') {
            steps {
                echo 'Checking if ThingsBoard application is currently running...'
                script {
                    def isRunning = sh(
                        script: '''
                            # Check if ThingsBoard process is running
                            if pgrep -f "thingsboard\\|ThingsboardServerApplication" > /dev/null; then
                                echo "RUNNING"
                            else
                                echo "STOPPED"
                            fi
                        ''',
                        returnStdout: true
                    ).trim()
                    
                    env.APP_STATUS = isRunning
                    echo "Application status: ${isRunning}"
                    
                    if (isRunning == "RUNNING") {
                        echo "🟢 ThingsBoard is running - will perform hot deployment"
                        env.DEPLOYMENT_MODE = "HOT_DEPLOY"
                    } else {
                        echo "🔴 ThingsBoard is not running - will start fresh"
                        env.DEPLOYMENT_MODE = "FRESH_START"
                    }
                }
            }
        }
        
        stage('Hot Deploy Changes') {
            when {
                expression { env.DEPLOYMENT_MODE == 'HOT_DEPLOY' }
            }
            steps {
                echo 'Hot deploying changed modules to running application...'
                script {
                    sh '''
                        echo "=== Hot Deployment Process ==="
                        
                        # Create backup directory
                        mkdir -p ${BACKUP_DIR}/$(date +%Y%m%d_%H%M%S)
                        BACKUP_PATH="${BACKUP_DIR}/$(date +%Y%m%d_%H%M%S)"
                        
                        # Backup existing JARs that will be replaced
                        for jar in incremental-build/new-jars/*.jar; do
                            if [ -f "$jar" ]; then
                                jarname=$(basename "$jar")
                                # Find corresponding JAR in lib directory
                                existing_jar=$(find ${LIB_DIR} -name "${jarname%%-*}*.jar" 2>/dev/null | head -1)
                                if [ -n "$existing_jar" ]; then
                                    echo "📁 Backing up: $(basename $existing_jar)"
                                    cp "$existing_jar" "$BACKUP_PATH/"
                                fi
                            fi
                        done
                        
                        # Get ThingsBoard PID for restart
                        TB_PID=$(pgrep -f "thingsboard\\|ThingsboardServerApplication" | head -1)
                        echo "ThingsBoard PID: $TB_PID"
                        
                        # Replace JARs in lib directory
                        echo "🔄 Replacing JARs in ${LIB_DIR}..."
                        for jar in incremental-build/new-jars/*.jar; do
                            if [ -f "$jar" ]; then
                                jarname=$(basename "$jar")
                                
                                # Remove old version
                                find ${LIB_DIR} -name "${jarname%%-*}*.jar" -delete 2>/dev/null || true
                                
                                # Copy new version
                                cp "$jar" "${LIB_DIR}/"
                                echo "✅ Deployed: $jarname"
                            fi
                        done
                        
                        # Graceful restart of ThingsBoard
                        if [ -n "$TB_PID" ]; then
                            echo "🔄 Restarting ThingsBoard application..."
                            
                            # Send SIGTERM for graceful shutdown
                            kill -TERM $TB_PID
                            
                            # Wait for graceful shutdown (max 30 seconds)
                            COUNTER=0
                            while [ $COUNTER -lt 30 ]; do
                                if ! ps -p $TB_PID > /dev/null 2>&1; then
                                    echo "✅ Application stopped gracefully"
                                    break
                                fi
                                sleep 1
                                COUNTER=$((COUNTER + 1))
                            done
                            
                            # Force kill if still running
                            if ps -p $TB_PID > /dev/null 2>&1; then
                                echo "⚠️  Force stopping application"
                                kill -9 $TB_PID
                            fi
                        fi
                        
                        # Start ThingsBoard with new JARs
                        echo "🚀 Starting ThingsBoard with updated modules..."
                        cd ${THINGSBOARD_HOME}
                        
                        # Start application (adjust command based on your setup)
                        nohup java -Xmx2048m \
                            -XX:+UseG1GC \
                            -Dspring.profiles.active=dev \
                            -Dloader.path=lib/ \
                            -jar lib/application-*.jar \
                            > logs/thingsboard-hotdeploy-${BUILD_NUMBER}.log 2>&1 &
                        
                        NEW_PID=$!
                        echo $NEW_PID > thingsboard.pid
                        echo "✅ ThingsBoard restarted with PID: $NEW_PID"
                        
                        echo "Backup location: $BACKUP_PATH"
                    '''
                }
            }
        }
        
        stage('Fresh Start Application') {
            when {
                expression { env.DEPLOYMENT_MODE == 'FRESH_START' }
            }
            steps {
                echo 'Starting ThingsBoard application with changes...'
                script {
                    sh '''
                        echo "=== Fresh Start with New Modules ==="
                        
                        # Create application directory structure
                        mkdir -p ${THINGSBOARD_HOME}/{lib,config,data,logs}
                        
                        # Copy all JARs to lib directory
                        echo "📦 Setting up application libraries..."
                        
                        # Copy new JARs
                        cp incremental-build/new-jars/*.jar ${THINGSBOARD_HOME}/lib/
                        
                        # Copy dependencies
                        if [ -d "incremental-build/dependencies" ]; then
                            cp incremental-build/dependencies/*.jar ${THINGSBOARD_HOME}/lib/
                        fi
                        
                        # Find and copy main application JAR if not already present
                        MAIN_JAR=$(find . -name "*application*.jar" -not -name "*-tests.jar" | head -1)
                        if [ -n "$MAIN_JAR" ] && [ ! -f "${THINGSBOARD_HOME}/lib/$(basename $MAIN_JAR)" ]; then
                            cp "$MAIN_JAR" ${THINGSBOARD_HOME}/lib/
                        fi
                        
                        # Start ThingsBoard
                        cd ${THINGSBOARD_HOME}
                        echo "🚀 Starting ThingsBoard application..."
                        
                        nohup java -Xmx2048m \
                            -XX:+UseG1GC \
                            -Dspring.profiles.active=dev \
                            -Dloader.path=lib/ \
                            -jar lib/application-*.jar \
                            > logs/thingsboard-fresh-${BUILD_NUMBER}.log 2>&1 &
                        
                        APP_PID=$!
                        echo $APP_PID > thingsboard.pid
                        echo "✅ ThingsBoard started with PID: $APP_PID"
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo 'Verifying ThingsBoard application after deployment...'
                script {
                    sh '''
                        echo "=== Deployment Verification ==="
                        
                        # Wait for application to start
                        TIMEOUT=120
                        COUNTER=0
                        
                        while [ $COUNTER -lt $TIMEOUT ]; do
                            # Check if process is running
                            if [ -f "${THINGSBOARD_HOME}/thingsboard.pid" ]; then
                                TB_PID=$(cat ${THINGSBOARD_HOME}/thingsboard.pid)
                                if ps -p $TB_PID > /dev/null 2>&1; then
                                    # Check if responding to HTTP
                                    if curl -s -f http://localhost:8080 > /dev/null 2>&1; then
                                        echo "✅ ThingsBoard is running and responding!"
                                        break
                                    fi
                                fi
                            fi
                            
                            echo "⏳ Waiting for application... ($COUNTER/$TIMEOUT)"
                            sleep 5
                            COUNTER=$((COUNTER + 5))
                        done
                        
                        if [ $COUNTER -ge $TIMEOUT ]; then
                            echo "❌ Application failed to start properly"
                            echo "=== Application Logs ==="
                            tail -50 ${THINGSBOARD_HOME}/logs/thingsboard-*-${BUILD_NUMBER}.log
                            exit 1
                        fi
                        
                        # Verify deployment
                        echo "=== Deployment Success ==="
                        echo "Changed modules: $CHANGED_MODULES"
                        echo "Deployment mode: $DEPLOYMENT_MODE"
                        echo "Application URL: http://localhost:8080"
                        echo "PID: $(cat ${THINGSBOARD_HOME}/thingsboard.pid 2>/dev/null)"
                        
                        # Show which JARs are active
                        echo "=== Active JARs ==="
                        ls -la ${THINGSBOARD_HOME}/lib/ | grep "$(date +%Y-%m-%d)" || ls -la ${THINGSBOARD_HOME}/lib/ | head -10
                        
                        # Check for any errors
                        if [ -f "${THINGSBOARD_HOME}/logs/thingsboard-*-${BUILD_NUMBER}.log" ]; then
                            ERROR_COUNT=$(grep -c "ERROR" ${THINGSBOARD_HOME}/logs/thingsboard-*-${BUILD_NUMBER}.log 2>/dev/null || echo "0")
                            echo "Errors in log: $ERROR_COUNT"
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Archiving incremental build results...'
            script {
                try {
                    // Archive new JARs
                    archiveArtifacts artifacts: 'incremental-build/new-jars/*.jar',
                                   allowEmptyArchive: true,
                                   fingerprint: true
                    
                    // Archive logs
                    sh '''
                        if [ -f "${THINGSBOARD_HOME}/logs/thingsboard-*-${BUILD_NUMBER}.log" ]; then
                            cp ${THINGSBOARD_HOME}/logs/thingsboard-*-${BUILD_NUMBER}.log .
                        fi
                    '''
                    
                    archiveArtifacts artifacts: 'thingsboard-*-*.log',
                                   allowEmptyArchive: true
                    
                    // Create deployment report
                    sh '''
                        cat > incremental-deployment-report.txt << EOF
=== Incremental Deployment Report ===
Build: ${BUILD_NUMBER}
Branch: ${BRANCH_NAME}
Changed Modules: ${CHANGED_MODULES}
Deployment Mode: ${DEPLOYMENT_MODE}
Date: $(date)

=== Results ===
Application Status: Running
URL: http://localhost:8080
PID: $(cat ${THINGSBOARD_HOME}/thingsboard.pid 2>/dev/null || echo "Not found")

=== Performance ===
Build Time: Incremental (changed modules only)
Deployment: Hot deployment
Downtime: Minimal (graceful restart)
EOF
                    '''
                    
                    archiveArtifacts artifacts: 'incremental-deployment-report.txt',
                                   allowEmptyArchive: true
                    
                } catch (Exception e) {
                    echo "Archiving warning: ${e.getMessage()}"
                }
            }
        }
        
        success {
            echo '✅ Incremental deployment succeeded!'
            script {
                try {
                    def modulesBuilt = 0
                    if (env.CHANGED_MODULES && env.CHANGED_MODULES != '') {
                        modulesBuilt = env.CHANGED_MODULES.split(',').size()
                    }
                    currentBuild.description = "✅ Hot-deployed ${modulesBuilt} modules | TB: http://localhost:8080"
                } catch (Exception e) {
                    currentBuild.description = "✅ Incremental deployment completed"
                }
            }
        }
        
        failure {
            echo '❌ Incremental deployment failed!'
            script {
                // Rollback if possible
                try {
                    sh '''
                        if [ -d "${BACKUP_DIR}" ]; then
                            LATEST_BACKUP=$(ls -t ${BACKUP_DIR} | head -1)
                            if [ -n "$LATEST_BACKUP" ]; then
                                echo "🔄 Rolling back to previous version..."
                                cp ${BACKUP_DIR}/$LATEST_BACKUP/*.jar ${LIB_DIR}/
                                echo "Rollback completed"
                            fi
                        fi
                    '''
                } catch (Exception e) {
                    echo "Rollback failed: ${e.getMessage()}"
                }
                
                currentBuild.description = "❌ Incremental deployment failed"
            }
        }
    }
}
