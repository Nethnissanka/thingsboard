pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx3072m -XX:+UseG1GC -XX:+UseStringDeduplication'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
        
        // Use Jenkins workspace instead of /opt (no permission issues)
        THINGSBOARD_HOME = "${WORKSPACE}/thingsboard-app"
        LIB_DIR = "${WORKSPACE}/thingsboard-app/lib"
        BACKUP_DIR = "${WORKSPACE}/thingsboard-app/backup"
        TARGET_BRANCH = 'pipeline'
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
                    echo "🌿 Detected branch: ${branchName}"
                }
                
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
                    
                    echo "📍 Current commit: ${env.CURRENT_COMMIT[0..7]}"
                    echo "📍 Previous commit: ${env.PREVIOUS_COMMIT[0..7]}"
                }
            }
        }
        
        stage('Detect Changed Modules') {
            steps {
                echo '🔍 Analyzing changed modules for incremental build...'
                script {
                    def changedModules = []
                    
                    try {
                        if (env.PREVIOUS_COMMIT != "initial") {
                            def changedFiles = sh(
                                script: "git diff --name-only ${env.PREVIOUS_COMMIT}..${env.CURRENT_COMMIT}",
                                returnStdout: true
                            ).trim()
                            
                            if (!changedFiles) {
                                echo "⚠️ No files changed between commits. Checking git log..."
                                changedFiles = sh(
                                    script: "git diff --name-only HEAD~1 HEAD",
                                    returnStdout: true
                                ).trim()
                            }
                            
                            def fileList = changedFiles.split('\n').findAll { it && it.trim() }
                            echo "📝 Changed files: ${fileList.join(', ')}"
                            
                            // Define ThingsBoard modules with proper paths
                            def moduleMap = [
                                'application': 'application',
                                'common': 'common', 
                                'dao': 'dao',
                                'rule-engine': 'rule-engine',
                                'transport': 'transport',
                                'ui-ngx': 'ui-ngx',
                                'tools': 'tools',
                                'netty-mqtt': 'netty-mqtt',
                                'rest-client': 'rest-client',
                                'monitoring': 'monitoring'
                            ]
                            
                            def moduleSet = [] as Set
                            
                            fileList.each { file ->
                                def parts = file.split('/')
                                if (parts.length > 0) {
                                    def topDir = parts[0]
                                    
                                    // Check if it's a known module
                                    if (moduleMap.containsKey(topDir)) {
                                        // Verify pom.xml exists
                                        def pomExists = sh(
                                            script: "test -f '${topDir}/pom.xml' && echo 'true' || echo 'false'",
                                            returnStdout: true
                                        ).trim() == 'true'
                                        
                                        if (pomExists) {
                                            moduleSet.add(topDir)
                                            echo "✅ Module found: ${topDir}"
                                        }
                                    }
                                }
                            }
                            
                            changedModules = moduleSet as List
                        }
                        
                        if (changedModules.isEmpty()) {
                            echo "⚠️ No Maven modules detected in changes."
                            echo "🔍 Checking for any Java files that might need compilation..."
                            
                            // Check if any Java files changed
                            def javaFiles = sh(
                                script: "git diff --name-only ${env.PREVIOUS_COMMIT}..${env.CURRENT_COMMIT} | grep '\\.java\$' || true",
                                returnStdout: true
                            ).trim()
                            
                            if (javaFiles) {
                                echo "☕ Java files changed, will build application module by default"
                                changedModules = ['application']
                            } else {
                                echo "📋 Only non-Java files changed (Jenkinsfile, configs, etc.)"
                                echo "🚀 Will deploy without building new JARs"
                                env.CHANGED_MODULES = 'NONE'
                                return
                            }
                        }
                        
                        env.CHANGED_MODULES = changedModules.join(',')
                        echo "🎯 Modules for incremental build: ${env.CHANGED_MODULES}"
                        
                    } catch (Exception e) {
                        echo "❌ Error detecting changed modules: ${e.getMessage()}"
                        echo "🔄 Fallback: Building application module"
                        env.CHANGED_MODULES = 'application'
                    }
                }
            }
        }
        
        stage('Build Only Changed Modules') {
            when {
                expression { env.CHANGED_MODULES && env.CHANGED_MODULES != '' && env.CHANGED_MODULES != 'NONE' }
            }
            steps {
                echo '🔨 Building ONLY changed modules (incremental)...'
                timeout(time: 45, unit: 'MINUTES') {
                    sh '''
                        echo "=== Incremental Build: Changed Modules Only ==="
                        
                        # Create staging directories
                        mkdir -p incremental-build/{new-jars,dependencies,logs}
                        
                        # Build each changed module
                        SUCCESS_COUNT=0
                        TOTAL_MODULES=$(echo $CHANGED_MODULES | tr ',' ' ' | wc -w)
                        
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            echo ""
                            echo "🏗️  Building module: $module ($((SUCCESS_COUNT + 1))/$TOTAL_MODULES)"
                            echo "====================================="
                            
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                cd "$module"
                                
                                # Clean build for this module only
                                echo "📦 Running: mvn clean package for $module"
                                
                                mvn clean package \
                                    -DskipTests=true \
                                    -Dmaven.test.skip=true \
                                    -Dmaven.javadoc.skip=true \
                                    -Dmaven.source.skip=true \
                                    -Dcheckstyle.skip=true \
                                    -Dspotbugs.skip=true \
                                    -Dpmd.skip=true \
                                    -Dfindbugs.skip=true \
                                    -Denforcer.skip=true \
                                    -Dmaven.compile.fork=true \
                                    -T 1C > "../incremental-build/logs/build-${module}.log" 2>&1
                                
                                if [ $? -eq 0 ]; then
                                    echo "✅ Build successful for: $module"
                                    
                                    # Copy built JARs
                                    JAR_COUNT=0
                                    find target -name "*.jar" -not -name "*-tests.jar" -not -name "*-sources.jar" -not -name "*-javadoc.jar" | while read jar; do
                                        if [ -f "$jar" ]; then
                                            cp "$jar" "../incremental-build/new-jars/"
                                            echo "  📄 Copied: $(basename $jar)"
                                            JAR_COUNT=$((JAR_COUNT + 1))
                                        fi
                                    done
                                    
                                    SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
                                else
                                    echo "❌ Build failed for: $module"
                                    echo "📋 Check log: incremental-build/logs/build-${module}.log"
                                    exit 1
                                fi
                                
                                cd ..
                            else
                                echo "⚠️  Module directory or pom.xml not found: $module"
                            fi
                        done
                        
                        echo ""
                        echo "=== Build Results ==="
                        echo "✅ Successfully built: $SUCCESS_COUNT/$TOTAL_MODULES modules"
                        echo "📦 Generated JARs:"
                        ls -la incremental-build/new-jars/
                        echo "📊 Total new JARs: $(ls incremental-build/new-jars/*.jar 2>/dev/null | wc -l)"
                    '''
                }
            }
        }
        
        stage('Install Changed Modules') {
            when {
                expression { env.CHANGED_MODULES && env.CHANGED_MODULES != '' && env.CHANGED_MODULES != 'NONE' }
            }
            steps {
                echo '📦 Installing changed modules to local Maven repository...'
                timeout(time: 15, unit: 'MINUTES') {
                    sh '''
                        echo "=== Installing Changed Modules ==="
                        
                        for module in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            if [ -d "$module" ] && [ -f "$module/pom.xml" ]; then
                                echo "🔧 Installing: $module"
                                
                                cd "$module"
                                mvn install -DskipTests=true -q
                                if [ $? -eq 0 ]; then
                                    echo "✅ Installed: $module"
                                else
                                    echo "❌ Install failed: $module"
                                    exit 1
                                fi
                                cd ..
                            fi
                        done
                        
                        echo ""
                        echo "=== Root Project Dependencies ==="
                        # Resolve dependencies for root project (fast)
                        mvn dependency:resolve -q
                        echo "✅ Dependencies resolved"
                    '''
                }
            }
        }
        
        stage('Setup Application Environment') {
            steps {
                echo '🏗️  Setting up ThingsBoard application environment...'
                script {
                    sh '''
                        echo "=== Application Environment Setup ==="
                        
                        # Create application directory structure
                        mkdir -p ${THINGSBOARD_HOME}/{lib,config,data,logs,backup}
                        
                        # Check if we have existing application
                        EXISTING_JARS=$(find ${LIB_DIR} -name "*.jar" 2>/dev/null | wc -l)
                        
                        if [ $EXISTING_JARS -eq 0 ]; then
                            echo "🆕 No existing application - setting up fresh installation"
                            
                            # Copy all required JARs from Maven repository
                            echo "📦 Copying dependencies from Maven repository..."
                            
                            # Find main application JAR in current build
                            MAIN_JAR=$(find . -name "*application*.jar" -not -name "*-tests.jar" -not -name "*-sources.jar" | head -1)
                            if [ -n "$MAIN_JAR" ]; then
                                cp "$MAIN_JAR" ${LIB_DIR}/
                                echo "✅ Main application JAR: $(basename $MAIN_JAR)"
                            fi
                            
                            # Copy essential dependencies
                            mvn dependency:copy-dependencies \
                                -DoutputDirectory=${LIB_DIR} \
                                -DincludeScope=runtime \
                                -DexcludeScope=test \
                                -q
                            
                            echo "📊 Total JARs in lib: $(ls ${LIB_DIR}/*.jar 2>/dev/null | wc -l)"
                        else
                            echo "♻️  Existing application found with $EXISTING_JARS JARs"
                        fi
                        
                        env.SETUP_MODE = $EXISTING_JARS > 0 ? "UPDATE" : "FRESH"
                    '''
                    
                    // Set setup mode in Jenkins environment
                    def libCount = sh(
                        script: "find ${env.LIB_DIR} -name '*.jar' 2>/dev/null | wc -l",
                        returnStdout: true
                    ).trim().toInteger()
                    
                    env.SETUP_MODE = libCount > 0 ? "UPDATE" : "FRESH"
                    echo "🔧 Setup mode: ${env.SETUP_MODE}"
                }
            }
        }
        
        stage('Deploy Changed Modules') {
            when {
                expression { env.CHANGED_MODULES && env.CHANGED_MODULES != 'NONE' }
            }
            steps {
                echo '🚀 Deploying changed modules to application...'
                script {
                    sh '''
                        echo "=== Hot Deployment Process ==="
                        
                        # Create backup before deployment
                        BACKUP_PATH="${BACKUP_DIR}/backup-$(date +%Y%m%d_%H%M%S)"
                        mkdir -p "$BACKUP_PATH"
                        
                        # Check if ThingsBoard is running
                        TB_PID=$(pgrep -f "thingsboard\\|ThingsboardServerApplication" | head -1)
                        
                        if [ -n "$TB_PID" ]; then
                            echo "🟢 ThingsBoard is running (PID: $TB_PID) - preparing hot deployment"
                            
                            # Backup existing JARs that will be replaced
                            for new_jar in incremental-build/new-jars/*.jar; do
                                if [ -f "$new_jar" ]; then
                                    jar_name=$(basename "$new_jar")
                                    # Find similar JAR in lib directory
                                    existing_jar=$(find ${LIB_DIR} -name "${jar_name%%-*}*.jar" 2>/dev/null | head -1)
                                    if [ -n "$existing_jar" ]; then
                                        cp "$existing_jar" "$BACKUP_PATH/"
                                        echo "💾 Backed up: $(basename $existing_jar)"
                                    fi
                                fi
                            done
                            
                            # Replace JARs
                            echo "🔄 Replacing JARs..."
                            for new_jar in incremental-build/new-jars/*.jar; do
                                if [ -f "$new_jar" ]; then
                                    jar_name=$(basename "$new_jar")
                                    
                                    # Remove old version(s)
                                    find ${LIB_DIR} -name "${jar_name%%-*}*.jar" -delete 2>/dev/null || true
                                    
                                    # Copy new version
                                    cp "$new_jar" "${LIB_DIR}/"
                                    echo "✅ Deployed: $jar_name"
                                fi
                            done
                            
                            echo "🔄 Restarting ThingsBoard gracefully..."
                            # Graceful shutdown
                            kill -TERM $TB_PID
                            
                            # Wait for shutdown
                            COUNTER=0
                            while [ $COUNTER -lt 30 ]; do
                                if ! ps -p $TB_PID > /dev/null 2>&1; then
                                    echo "✅ Graceful shutdown completed"
                                    break
                                fi
                                sleep 1
                                COUNTER=$((COUNTER + 1))
                            done
                            
                            # Force kill if necessary
                            if ps -p $TB_PID > /dev/null 2>&1; then
                                kill -9 $TB_PID
                                echo "⚡ Force stopped"
                            fi
                            
                        else
                            echo "🔴 ThingsBoard not running - will start fresh"
                            
                            # Copy new JARs
                            for new_jar in incremental-build/new-jars/*.jar; do
                                if [ -f "$new_jar" ]; then
                                    cp "$new_jar" "${LIB_DIR}/"
                                    echo "✅ Added: $(basename $new_jar)"
                                fi
                            done
                        fi
                        
                        # Start ThingsBoard
                        cd ${THINGSBOARD_HOME}
                        echo "🚀 Starting ThingsBoard with updated modules..."
                        
                        # Find main application JAR
                        MAIN_JAR=$(find lib -name "*application*.jar" | head -1)
                        if [ -z "$MAIN_JAR" ]; then
                            echo "❌ No main application JAR found in lib directory"
                            exit 1
                        fi
                        
                        # Start application
                        nohup java -Xmx2048m \
                            -XX:+UseG1GC \
                            -XX:+HeapDumpOnOutOfMemoryError \
                            -Dspring.profiles.active=dev \
                            -Dloader.path=lib/ \
                            -jar "$MAIN_JAR" \
                            > logs/thingsboard-${BUILD_NUMBER}.log 2>&1 &
                        
                        NEW_PID=$!
                        echo $NEW_PID > thingsboard.pid
                        echo "✅ ThingsBoard started with PID: $NEW_PID"
                        echo "📍 Backup location: $BACKUP_PATH"
                        
                        # Save deployment info
                        echo "Deployment completed at $(date)" > deployment-info.txt
                        echo "Changed modules: $CHANGED_MODULES" >> deployment-info.txt
                        echo "PID: $NEW_PID" >> deployment-info.txt
                        echo "Backup: $BACKUP_PATH" >> deployment-info.txt
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo '🔍 Verifying ThingsBoard deployment...'
                timeout(time: 3, unit: 'MINUTES') {
                    script {
                        sh '''
                            echo "=== Deployment Verification ==="
                            
                            # Wait for application startup
                            TIMEOUT=120
                            COUNTER=0
                            
                            while [ $COUNTER -lt $TIMEOUT ]; do
                                if [ -f "${THINGSBOARD_HOME}/thingsboard.pid" ]; then
                                    TB_PID=$(cat ${THINGSBOARD_HOME}/thingsboard.pid)
                                    
                                    if ps -p $TB_PID > /dev/null 2>&1; then
                                        echo "✅ Process is running (PID: $TB_PID)"
                                        
                                        # Check if application port is listening
                                        if netstat -tln | grep ":8080 " > /dev/null 2>&1; then
                                            echo "✅ ThingsBoard is listening on port 8080"
                                            
                                            # Try HTTP health check
                                            if curl -s -f -m 10 http://localhost:8080/actuator/health > /dev/null 2>&1; then
                                                echo "✅ Health check passed!"
                                                break
                                            elif curl -s -f -m 10 http://localhost:8080 > /dev/null 2>&1; then
                                                echo "✅ HTTP response received!"
                                                break
                                            fi
                                        fi
                                    fi
                                fi
                                
                                echo "⏳ Waiting for startup... ($COUNTER/$TIMEOUT seconds)"
                                sleep 5
                                COUNTER=$((COUNTER + 5))
                            done
                            
                            if [ $COUNTER -ge $TIMEOUT ]; then
                                echo "❌ Application startup verification failed"
                                echo ""
                                echo "=== Troubleshooting Info ==="
                                
                                if [ -f "${THINGSBOARD_HOME}/thingsboard.pid" ]; then
                                    TB_PID=$(cat ${THINGSBOARD_HOME}/thingsboard.pid)
                                    if ps -p $TB_PID > /dev/null 2>&1; then
                                        echo "✅ Process is still running (PID: $TB_PID)"
                                    else
                                        echo "❌ Process died (PID: $TB_PID)"
                                    fi
                                fi
                                
                                echo ""
                                echo "=== Recent Logs ==="
                                tail -30 ${THINGSBOARD_HOME}/logs/thingsboard-${BUILD_NUMBER}.log || echo "No logs found"
                                
                                # Don't fail the build, just warn
                                echo "⚠️  Deployment completed but verification failed - check logs"
                            else
                                echo ""
                                echo "=== Deployment Success ==="
                                echo "🎯 Changed modules: $CHANGED_MODULES"
                                echo "🚀 Application URL: http://localhost:8080"
                                echo "🆔 PID: $(cat ${THINGSBOARD_HOME}/thingsboard.pid)"
                                echo "📊 Active JARs: $(ls ${LIB_DIR}/*.jar | wc -l)"
                                echo "⏱️  Verification time: $COUNTER seconds"
                            fi
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo '📦 Archiving build artifacts...'
            script {
                try {
                    // Archive new JARs
                    archiveArtifacts artifacts: 'incremental-build/new-jars/*.jar',
                                   allowEmptyArchive: true,
                                   fingerprint: true
                    
                    // Archive build logs
                    archiveArtifacts artifacts: 'incremental-build/logs/*.log',
                                   allowEmptyArchive: true
                    
                    // Archive application logs
                    sh '''
                        if [ -f "${THINGSBOARD_HOME}/logs/thingsboard-${BUILD_NUMBER}.log" ]; then
                            cp ${THINGSBOARD_HOME}/logs/thingsboard-${BUILD_NUMBER}.log .
                        fi
                    '''
                    
                    archiveArtifacts artifacts: 'thingsboard-*.log',
                                   allowEmptyArchive: true
                    
                    // Archive deployment info
                    archiveArtifacts artifacts: 'deployment-info.txt',
                                   allowEmptyArchive: true
                    
                } catch (Exception e) {
                    echo "⚠️ Archiving warning: ${e.getMessage()}"
                }
            }
        }
        
        success {
            echo '✅ Incremental deployment completed successfully!'
            script {
                try {
                    def modulesBuilt = 0
                    if (env.CHANGED_MODULES && env.CHANGED_MODULES != '' && env.CHANGED_MODULES != 'NONE') {
                        modulesBuilt = env.CHANGED_MODULES.split(',').size()
                    }
                    
                    if (modulesBuilt > 0) {
                        currentBuild.description = "✅ Built & deployed ${modulesBuilt} modules: ${env.CHANGED_MODULES}"
                    } else {
                        currentBuild.description = "✅ No code changes - deployment skipped"
                    }
                } catch (Exception e) {
                    currentBuild.description = "✅ Incremental deployment completed"
                }
            }
        }
        
        failure {
            echo '❌ Incremental deployment failed!'
            script {
                currentBuild.description = "❌ Failed: ${env.CHANGED_MODULES ?: 'Unknown modules'}"
                
                // Attempt rollback
                try {
                    sh '''
                        if [ -d "${BACKUP_DIR}" ]; then
                            LATEST_BACKUP=$(ls -t ${BACKUP_DIR} | head -1)
                            if [ -n "$LATEST_BACKUP" ] && [ -d "${BACKUP_DIR}/$LATEST_BACKUP" ]; then
                                echo "🔄 Attempting rollback..."
                                cp ${BACKUP_DIR}/$LATEST_BACKUP/*.jar ${LIB_DIR}/ 2>/dev/null || true
                                echo "Rollback attempted - check application manually"
                            fi
                        fi
                    '''
                } catch (Exception e) {
                    echo "Rollback failed: ${e.getMessage()}"
                }
            }
        }
        
        unstable {
            echo '⚠️ Build unstable but deployment may have succeeded'
            currentBuild.description = "⚠️ Unstable: ${env.CHANGED_MODULES ?: 'Check logs'}"
        }
    }
}
