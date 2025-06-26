/* ------------------------------------------------------------------------- */
/* ThingsBoard CI/CD – Incremental Hot‑Deploy Pipeline                       */
/* ------------------------------------------------------------------------- */
pipeline {
    agent any

    environment {
        MAVEN_OPTS             = '-Xmx3072m -XX:+UseG1GC -XX:+UseStringDeduplication'
        JAVA_HOME              = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME                = '/usr/share/maven'
        PATH                   = "${env.PATH}:/usr/bin"

        TARGET_BRANCH          = 'pipeline'
        BUILD_PROFILE          = 'fast-build'

        THINGSBOARD_HOME       = '/home/nethmi/Projects/thingsboard'
        THINGSBOARD_PORT       = '8080'
        THINGSBOARD_PID_FILE   = '/tmp/thingsboard.pid'
    }

    triggers { githubPush() }

    /* ===================================================================== */
    /*  1.  Validate branch                                                  */
    /* ===================================================================== */
    stages {
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
                    echo "Target branch  : ${TARGET_BRANCH}"
                    if (b && b != TARGET_BRANCH) {
                        echo "⚠️  Pipeline tuned for '${TARGET_BRANCH}' but running on '${b}'"
                    }
                }
            }
        }

    /* ===================================================================== */
    /*  2.  Checkout + capture commits                                       */
    /* ===================================================================== */
        stage('Checkout & Setup') {
            steps {
                checkout scm
                script {
                    env.CURRENT_COMMIT  = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    env.PREVIOUS_COMMIT = sh(script: 'git rev-parse HEAD~1 || echo initial',
                                             returnStdout: true).trim()
                }
                echo "Current commit : ${env.CURRENT_COMMIT}"
                echo "Previous commit: ${env.PREVIOUS_COMMIT}"
            }
        }

    /* ===================================================================== */
    /*  3.  Detect changed modules                                           */
    /* ===================================================================== */
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
                                def hasPom = sh(script: "test -f '${top}/pom.xml' && echo true || echo false",
                                                returnStdout: true).trim() == 'true'
                                if (hasPom) set << top
                            }
                        }
                        changedModules = set as List
                    } else {
                        echo 'Initial build – mark as full‑build'
                        changedModules = ['full-build']
                    }

                    if (changedModules.isEmpty()) {
                        echo '❌ No changed modules detected – finishing early.'
                        currentBuild.result = 'NOT_BUILT'
                        error 'Nothing to build'
                    }

                    env.CHANGED_MODULES = changedModules.join(',')
                    echo "Modules that changed (pre‑processing): ${env.CHANGED_MODULES}"

                    /* Auto‑add application when a dependency changed */
                    if (env.CHANGED_MODULES != 'full-build') {
                        def appDeps = [
                            'common','dao','transport','rest-client',
                            'rule-engine','tools','monitoring','netty-mqtt','ui-ngx'
                        ]
                        def list = env.CHANGED_MODULES.tokenize(',')
                        if (list.intersect(appDeps) && !list.contains('application')) {
                            list << 'application'
                            echo "📦 Added 'application' because a dependency changed."
                        }
                        env.CHANGED_MODULES = list.unique().join(',')
                    }
                    echo "🔄 Final module list: ${env.CHANGED_MODULES}"
                }
            }
        }

    /* ===================================================================== */
    /*  4.  Build info banner                                                */
    /* ===================================================================== */
        stage('Build Info') {
            steps {
                echo "Building ThingsBoard multi‑module project"
                echo "Branch         : ${env.CURRENT_BRANCH}"
                echo "Build #        : ${env.BUILD_NUMBER}"
                echo "Changed modules: ${env.CHANGED_MODULES}"
                sh 'java -version'
                sh 'mvn  -version'
            }
        }

    /* ===================================================================== */
    /*  5.  Check if TB is already running                                   */
    /* ===================================================================== */
        stage('Check Running Application') {
            steps {
                script {
                    env.APP_RUNNING = sh(
                        script: '''
                            if curl -s -f http://localhost:${THINGSBOARD_PORT} >/dev/null 2>&1; then
                                echo true
                            elif [ -f "${THINGSBOARD_PID_FILE}" ] && ps -p $(cat ${THINGSBOARD_PID_FILE}) >/dev/null 2>&1; then
                                echo true
                            else echo false; fi
                        ''', returnStdout: true).trim()

                    if (env.APP_RUNNING == 'true')
                        echo '✅ Application is running – hot‑deploy path.'
                    else {
                        echo '⚠️ No running app – full‑build path.'
                        env.CHANGED_MODULES = 'full-build'
                    }
                }
            }
        }

    /* ===================================================================== */
    /*  6.  HOT PATH – Build & install only changed modules                  */
    /* ===================================================================== */
        stage('Build Changed Modules Only') {
            when {
                expression {
                    env.CHANGED_MODULES &&
                    env.CHANGED_MODULES != 'full-build' &&
                    env.APP_RUNNING == 'true'
                }
            }
            steps {
                echo '🚀 Building changed modules…'
                timeout(time: 10, unit: 'MINUTES') {
                    sh '''
                        MODULE_LIST=""
                        for m in $(echo $CHANGED_MODULES | tr ',' ' '); do
                            [ -f "$m/pom.xml" ] && MODULE_LIST="$MODULE_LIST -pl $m"
                        done
                        echo "Maven reactor modules:$MODULE_LIST"
                        mvn package $MODULE_LIST -am -DskipTests -T 2C -q
                        mvn install  $MODULE_LIST -DskipTests -q
                    '''
                }
            }
        }

        /* ------------------------------------------------------------------ */
        /* Install Changed Modules step removed (merged above for brevity)     */
        /* ------------------------------------------------------------------ */

    /* ===================================================================== */
    /*  7.  HOT PATH – Redeploy                                              */
    /* ===================================================================== */
        stage('Hot Deploy to Running Application') {
            when {
                expression {
                    env.CHANGED_MODULES &&
                    env.CHANGED_MODULES != 'full-build' &&
                    env.APP_RUNNING == 'true'
                }
            }
            steps {
                echo '♻️  Hot‑deploying new Boot JAR…'
                script { sh '''
                        cd application
                        mvn package -DskipTests -q
                        cd ..

                        JAR="./application/target/thingsboard-*-boot.jar"
                        JAR=$(ls $JAR | head -1)

                        [ -f "$JAR" ] || { echo "❌ Boot jar not found"; exit 1; }

                        echo "Restarting TB with $JAR"
                        if [ -f "${THINGSBOARD_PID_FILE}" ]; then
                            kill $(cat ${THINGSBOARD_PID_FILE}) || true
                            sleep 5
                        fi
                        nohup java -Xmx2048m -jar "$JAR" \
                              > logs/thingsboard-restart-${BUILD_NUMBER}.log 2>&1 &
                        echo $! > ${THINGSBOARD_PID_FILE}
                    '''
                }
            }
        }

    /* ===================================================================== */
    /*  8.  FULL BUILD path                                                  */
    /* ===================================================================== */
        stage('Build Complete Application') {
            when {
                expression { env.CHANGED_MODULES == 'full-build' || env.APP_RUNNING != 'true' }
            }
            steps {
                echo '🛠️  Building full project…'
                timeout(time: 45, unit: 'MINUTES') {
                    sh '''
                        mvn clean package -DskipTests -T 2C
                    '''
                }
            }
        }

        stage('Start ThingsBoard Application') {
            when {
                expression { env.CHANGED_MODULES == 'full-build' || env.APP_RUNNING != 'true' }
            }
            steps {
                echo '🟢 Starting ThingsBoard…'
                script { sh '''
                        JAR=$(find ./application/target -name "*-boot.jar" | head -1)
                        [ -f "$JAR" ] || { echo "❌ Boot jar not found"; exit 1; }
                        [ -f "${THINGSBOARD_PID_FILE}" ] && kill $(cat ${THINGSBOARD_PID_FILE}) || true
                        nohup java -Xmx2048m -jar "$JAR" \
                              > logs/thingsboard-${BUILD_NUMBER}.log 2>&1 &
                        echo $! > ${THINGSBOARD_PID_FILE}
                    '''
                }
            }
        }

    /* ===================================================================== */
    /*  9.  Health check + Archive + Post                                    */
    /* ===================================================================== */

        stage('Application Health Check') {
            steps {
                sh '''
                    echo -n "Health check… "
                    curl -s -f http://localhost:${THINGSBOARD_PORT} >/dev/null && echo "OK" || echo "FAIL"
                '''
            }
        }

        stage('Archive Results') {
            steps {
                archiveArtifacts artifacts: 'application/target/*-boot.jar,logs/**/*.log',
                                 fingerprint: true, allowEmptyArchive: true
            }
        }
    }

    /* --------------------------------------------------------------------- */
    /* POST                                                                   */
    /* --------------------------------------------------------------------- */
    post {
        success { echo '✅ Build & deploy succeeded' }
        failure { echo '❌ Build or deploy failed'   }
        always  { echo 'Pipeline completed'         }
    }
}

