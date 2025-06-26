pipeline {
    agent any

    environment {
        MAVEN_OPTS = '-Xmx3072m -XX:+UseG1GC -XX:+UseStringDeduplication'
        JAVA_HOME  = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME    = '/usr/share/maven'
        PATH       = "${env.PATH}:/usr/bin"

        TARGET_BRANCH         = 'pipeline'
        BUILD_PROFILE         = 'fast-build'

        THINGSBOARD_HOME      = '/home/nethmi/Projects/thingsboard'
        THINGSBOARD_PORT      = '8080'
        THINGSBOARD_PID_FILE  = '/tmp/thingsboard.pid'
        THINGSBOARD_LOG_FILE  = '/tmp/thingsboard.log'
        BACKUP_DIR           = '/tmp/thingsboard-backup'
        DEPLOY_TIMEOUT       = '300' // 5 minutes
    }

    triggers { 
        githubPush() 
        pollSCM('H/5 * * * *') // Poll every 5 minutes as fallback
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        retry(1)
        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
    }

    stages {

    /* ------------------------------------------------------------------ */
    /* 1️⃣  Validate branch                                               */
    /* ------------------------------------------------------------------ */
        stage('Validate Branch') {
            steps {
                script {
                    def b = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''
                    if (!b) {
                        b = sh(script: 'git branch --show-current || git rev-parse --abbrev-ref HEAD',
                               returnStdout: true).trim()
                    }
                    if (b.startsWith('origin/')) b = b.substring(7)

                    env.CURRENT_BRANCH = b
                    echo "Detected branch: ${b}"
                    echo "Target branch : ${TARGET_BRANCH}"
                    if (b && b != TARGET_BRANCH) {
                        echo "⚠️  Pipeline was tuned for '${TARGET_BRANCH}', but running on '${b}'"
                    }
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 2️⃣  Checkout + capture HEAD and HEAD~1                            */
    /* ------------------------------------------------------------------ */
        stage('Checkout & Setup') {
            steps {
                checkout scm
                script {
                    env.CURRENT_COMMIT  = sh(script: 'git rev-parse HEAD',      returnStdout: true).trim()
                    env.PREVIOUS_COMMIT = sh(script: 'git rev-parse HEAD~1 || echo initial',
                                             returnStdout: true).trim()
                    
                    // Create backup directory if it doesn't exist
                    sh "mkdir -p ${BACKUP_DIR}"
                }
                echo "Current commit : ${env.CURRENT_COMMIT}"
                echo "Previous commit: ${env.PREVIOUS_COMMIT}"
            }
        }

    /* ------------------------------------------------------------------ */
    /* 3️⃣  Detect changed modules (with early‑exit + auto‑add app)       */
    /* ------------------------------------------------------------------ */
        stage('Detect Changed Modules') {
            steps {
                script {
                    def changedFiles   = []
                    def changedModules = []

                    if (env.PREVIOUS_COMMIT != 'initial') {
                        changedFiles = sh(
                            script: "git diff --name-only ${env.PREVIOUS_COMMIT}..${env.CURRENT_COMMIT}",
                            returnStdout: true
                        ).trim().split('\n').findAll { it }

                        echo "Changed files: ${changedFiles.join(', ')}"

                        def known = [
                            'application','common','dao','edqs','monitoring','netty-mqtt',
                            'packaging','rest-client','rule-engine','tools','transport','ui-ngx','msa'
                        ]

                        def set = [] as Set
                        changedFiles.each { f ->
                            def top = f.tokenize('/')[0]
                            if (known.contains(top)) {
                                def hasPom = sh(script:"test -f '${top}/pom.xml' && echo true || echo false",
                                                returnStdout:true).trim() == 'true'
                                if (hasPom) set << top
                            }
                        }
                        changedModules = set as List
                    } else {
                        echo 'Initial build – will build all modules'
                        changedModules = ['full-build']
                    }

                    /* Early exit when absolutely nothing changed */
                    if (changedModules.isEmpty()) {
                        echo '❌ No changed modules detected – finishing early.'
                        currentBuild.result = 'NOT_BUILT'
                        error 'Nothing to build'
                    }

                    env.CHANGED_MODULES = changedModules.join(',')
                    echo "Modules that changed (pre‑processing): ${env.CHANGED_MODULES}"

                    /* Force‑add `application` if any dependency changed */
                    if (env.CHANGED_MODULES != 'full-build') {
                        def appDeps = ['common','dao','transport','rest-client',
                                       'rule-engine','tools','monitoring','netty-mqtt','ui-ngx']
                        def list = env.CHANGED_MODULES.tokenize(',')
                        if (list.intersect(appDeps) && !list.contains('application')) {
                            list << 'application'
                            echo "📦 Added 'application' to changed modules because a dependency changed."
                        }
                        env.CHANGED_MODULES = list.unique().join(',')
                    }
                    echo "🔄 Final changed modules list: ${env.CHANGED_MODULES}"
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 4️⃣  Build‑info banner                                             */
    /* ------------------------------------------------------------------ */
        stage('Build Info') {
            steps {
                echo "Building ThingsBoard multi‑module project"
                echo "Branch         : ${env.CURRENT_BRANCH}"
                echo "Build #        : ${env.BUILD_NUMBER}"
                echo "Changed modules: ${env.CHANGED_MODULES}"
                sh 'java -version'
                sh 'mvn  -version'
                sh 'df -h .' // Show disk space
            }
        }

    /* ------------------------------------------------------------------ */
    /* 5️⃣  Check live application                                        */
    /* ------------------------------------------------------------------ */
        stage('Check Running Application') {
            steps {
                script {
                    env.APP_RUNNING = sh(
                        script: '''
                               if curl -s -f http://localhost:${THINGSBOARD_PORT}/api/noauth/healthcheck >/dev/null 2>&1; then
                                   echo true
                               elif [ -f "${THINGSBOARD_PID_FILE}" ] && ps -p $(cat ${THINGSBOARD_PID_FILE}) >/dev/null 2>&1; then
                                   echo true
                               else echo false; fi
                        ''', returnStdout: true).trim()

                    if (env.APP_RUNNING == 'true') {
                        echo '✅ Application is running – will hot‑deploy.'
                        // Backup current application before hot deploy
                        sh '''
                            if [ -f "${THINGSBOARD_HOME}/application/target/thingsboard-*.jar" ]; then
                                cp "${THINGSBOARD_HOME}/application/target/thingsboard-"*.jar "${BACKUP_DIR}/thingsboard-backup-$(date +%Y%m%d_%H%M%S).jar"
                                echo "📦 Created backup of current application"
                            fi
                        '''
                    } else {
                        echo '⚠️ Application is not running – will do full build.'
                        env.CHANGED_MODULES = 'full-build'
                    }
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 6️⃣  Build changed modules (hot path)                              */
    /* ------------------------------------------------------------------ */
        stage('Build Changed Modules Only') {
            when {
                expression {
                    env.CHANGED_MODULES &&
                    env.CHANGED_MODULES != 'full-build' &&
                    env.APP_RUNNING == 'true'
                }
            }
            steps {
                echo 'Building only changed modules…'
                timeout(time: 10, unit: 'MINUTES') {
                    sh '''
                        MODULE_LIST=""
                        for m in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            [ -f "$m/pom.xml" ] && MODULE_LIST="$MODULE_LIST -pl $m"
                        done
                        echo "Maven reactor modules: $MODULE_LIST"
                        mvn clean package $MODULE_LIST -am -DskipTests -T 2C -q -P${BUILD_PROFILE}
                    '''
                }
            }
            post {
                failure {
                    echo '❌ Hot build failed, falling back to full build'
                    script {
                        env.CHANGED_MODULES = 'full-build'
                        env.APP_RUNNING = 'false'
                    }
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 7️⃣  Install artifacts                                             */
    /* ------------------------------------------------------------------ */
        stage('Install Artifacts') {
            when {
                expression {
                    env.CHANGED_MODULES != 'full-build' &&
                    env.APP_RUNNING == 'true'
                }
            }
            steps {
                echo 'Installing changed modules to local repository…'
                timeout(time: 5, unit: 'MINUTES') {
                    sh '''
                        MODULE_LIST=""
                        for m in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            [ -f "$m/pom.xml" ] && MODULE_LIST="$MODULE_LIST -pl $m"
                        done
                        mvn install $MODULE_LIST -DskipTests -q
                    '''
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 8️⃣  Hot deploy                                                    */
    /* ------------------------------------------------------------------ */
        stage('Hot Deploy') {
            when {
                expression {
                    env.CHANGED_MODULES != 'full-build' &&
                    env.APP_RUNNING == 'true' &&
                    env.CHANGED_MODULES.contains('application')
                }
            }
            steps {
                echo '🔥 Hot deploying application…'
                timeout(time: 3, unit: 'MINUTES') {
                    sh '''
                        # Stop the application gracefully
                        if [ -f "${THINGSBOARD_PID_FILE}" ]; then
                            PID=$(cat ${THINGSBOARD_PID_FILE})
                            if ps -p $PID > /dev/null; then
                                echo "Stopping ThingsBoard (PID: $PID)"
                                kill $PID
                                # Wait for graceful shutdown
                                for i in {1..30}; do
                                    if ! ps -p $PID > /dev/null; then
                                        echo "Application stopped gracefully"
                                        break
                                    fi
                                    sleep 1
                                done
                                # Force kill if still running
                                if ps -p $PID > /dev/null; then
                                    echo "Force killing application"
                                    kill -9 $PID
                                fi
                            fi
                        fi
                        
                        # Copy new JAR
                        cd ${THINGSBOARD_HOME}
                        if [ -f "application/target/thingsboard-"*.jar ]; then
                            cp application/target/thingsboard-*.jar ${THINGSBOARD_HOME}/
                            echo "✅ New application JAR deployed"
                        else
                            echo "❌ No application JAR found"
                            exit 1
                        fi
                    '''
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 9️⃣  Full build (cold path)                                        */
    /* ------------------------------------------------------------------ */
        stage('Full Build') {
            when {
                expression { env.CHANGED_MODULES == 'full-build' }
            }
            steps {
                echo '🏗️ Performing full build…'
                timeout(time: 20, unit: 'MINUTES') {
                    sh '''
                        mvn clean install -DskipTests -T 2C -P${BUILD_PROFILE}
                    '''
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 🔟  Start application                                             */
    /* ------------------------------------------------------------------ */
        stage('Start Application') {
            when {
                expression { env.APP_RUNNING != 'true' }
            }
            steps {
                echo '🚀 Starting ThingsBoard application…'
                timeout(time: 5, unit: 'MINUTES') {
                    sh '''
                        cd ${THINGSBOARD_HOME}
                        
                        # Find the application JAR
                        APP_JAR=$(find . -name "thingsboard-*.jar" -not -path "./application/target/*" | head -1)
                        if [ -z "$APP_JAR" ]; then
                            APP_JAR=$(find application/target -name "thingsboard-*.jar" | head -1)
                        fi
                        
                        if [ -z "$APP_JAR" ]; then
                            echo "❌ No application JAR found"
                            exit 1
                        fi
                        
                        echo "Starting application: $APP_JAR"
                        nohup java -jar "$APP_JAR" > ${THINGSBOARD_LOG_FILE} 2>&1 &
                        echo $! > ${THINGSBOARD_PID_FILE}
                        echo "✅ Application started with PID: $(cat ${THINGSBOARD_PID_FILE})"
                    '''
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 1️⃣1️⃣  Health check                                               */
    /* ------------------------------------------------------------------ */
        stage('Health Check') {
            steps {
                echo '🏥 Performing health check…'
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def healthCheckPassed = false
                        def maxAttempts = 60
                        def attempt = 0
                        
                        while (!healthCheckPassed && attempt < maxAttempts) {
                            attempt++
                            try {
                                def response = sh(
                                    script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${THINGSBOARD_PORT}/api/noauth/healthcheck",
                                    returnStdout: true
                                ).trim()
                                
                                if (response == '200') {
                                    healthCheckPassed = true
                                    echo "✅ Health check passed on attempt ${attempt}"
                                } else {
                                    echo "⏳ Health check attempt ${attempt}/${maxAttempts}, got HTTP ${response}"
                                    sleep 5
                                }
                            } catch (Exception e) {
                                echo "⏳ Health check attempt ${attempt}/${maxAttempts}, connection failed"
                                sleep 5
                            }
                        }
                        
                        if (!healthCheckPassed) {
                            // Try to get application logs for debugging
                            sh "tail -50 ${THINGSBOARD_LOG_FILE} || echo 'No log file found'"
                            error "❌ Health check failed after ${maxAttempts} attempts"
                        }
                    }
                }
            }
        }

    /* ------------------------------------------------------------------ */
    /* 1️⃣2️⃣  Archive artifacts                                          */
    /* ------------------------------------------------------------------ */
        stage('Archive Artifacts') {
            steps {
                echo '📦 Archiving build artifacts…'
                script {
                    // Archive JARs
                    archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
                    
                    // Archive logs if they exist
                    if (fileExists(env.THINGSBOARD_LOG_FILE)) {
                        archiveArtifacts artifacts: env.THINGSBOARD_LOG_FILE, allowEmptyArchive: true
                    }
                }
            }
        }
    }

    /* ------------------------------------------------------------------ */
    /* Post-build actions                                                 */
    /* ------------------------------------------------------------------ */
    post {
        always {
            echo "Pipeline completed with status: ${currentBuild.result ?: 'SUCCESS'}"
            
            // Clean up old backups (keep last 5)
            sh '''
                if [ -d "${BACKUP_DIR}" ]; then
                    cd "${BACKUP_DIR}"
                    ls -t thingsboard-backup-*.jar 2>/dev/null | tail -n +6 | xargs rm -f
                    echo "Cleaned up old backup files"
                fi
            '''
        }
        
        success {
            echo '✅ Pipeline completed successfully!'
            // Send success notification if needed
            // slackSend channel: '#builds', color: 'good', message: "✅ ThingsBoard build #${BUILD_NUMBER} succeeded"
        }
        
        failure {
            echo '❌ Pipeline failed!'
            
            // Attempt rollback if we have a backup and the app was running before
            script {
                if (env.APP_RUNNING == 'true' && env.CHANGED_MODULES != 'full-build') {
                    echo '🔄 Attempting rollback to previous version…'
                    sh '''
                        if [ -f "${THINGSBOARD_PID_FILE}" ]; then
                            kill $(cat ${THINGSBOARD_PID_FILE}) 2>/dev/null || true
                        fi
                        
                        # Find the latest backup
                        LATEST_BACKUP=$(ls -t ${BACKUP_DIR}/thingsboard-backup-*.jar 2>/dev/null | head -1)
                        if [ -n "$LATEST_BACKUP" ]; then
                            echo "Rolling back to: $LATEST_BACKUP"
                            cp "$LATEST_BACKUP" ${THINGSBOARD_HOME}/thingsboard-rollback.jar
                            cd ${THINGSBOARD_HOME}
                            nohup java -jar thingsboard-rollback.jar > ${THINGSBOARD_LOG_FILE} 2>&1 &
                            echo $! > ${THINGSBOARD_PID_FILE}
                            echo "✅ Rollback completed"
                        else
                            echo "❌ No backup found for rollback"
                        fi
                    '''
                }
            }
            
            // Send failure notification
            // slackSend channel: '#builds', color: 'danger', message: "❌ ThingsBoard build #${BUILD_NUMBER} failed"
        }
        
        unstable {
            echo '⚠️ Pipeline completed with warnings'
        }
        
        cleanup {
            echo 'Performing cleanup tasks...'
            // Clean up workspace if needed
            // cleanWs()
        }
    }
}
