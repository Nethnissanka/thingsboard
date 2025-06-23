pipeline {
    agent any
    
    environment {
        MAVEN_OPTS = '-Xmx3072m -XX:+UseG1GC'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-2.el9.x86_64'
        M2_HOME = '/usr/share/maven'
        PATH = "${env.PATH}:/usr/bin"
        
        ENV_NAME = 'PRODUCTION'
        DEPLOY_TARGET = 'prod-server'
        BUILD_PROFILE = 'prod'
    }
    
    parameters {
        choice(
            name: 'RELEASE_BRANCH',
            choices: ['feature/*','master' ],
            description: 'Select release branch for production build'
        )
        string(
            name: 'VERSION_TAG',
            defaultValue: '',
            description: 'Version tag for this production release (e.g., v1.2.3)'
        )
        booleanParam(
            name: 'FORCE_FULL_BUILD',
            defaultValue: false,
            description: 'Force full build instead of incremental?'
        )
        booleanParam(
            name: 'CREATE_BACKUP',
            defaultValue: true,
            description: 'Create backup before production deployment?'
        )
        booleanParam(
            name: 'DEPLOY_TO_PROD',
            defaultValue: false,
            description: 'Deploy to production? (Requires approval)'
        )
    }
    
    stages {
        stage('Production Checkout') {
            steps {
                echo "🔒 Checking out source code for PRODUCTION build..."
                script {
                    if (params.VERSION_TAG) {
                        checkout([$class: 'GitSCM',
                                branches: [[name: "refs/tags/${params.VERSION_TAG}"]],
                                userRemoteConfigs: scm.userRemoteConfigs])
                    } else {
                        checkout([$class: 'GitSCM',
                                branches: [[name: "*/${params.RELEASE_BRANCH}"]],
                                userRemoteConfigs: scm.userRemoteConfigs])
                    }
                }
            }
        }
        
        stage('Detect Changed Modules') {
            steps {
                echo '🔍 Detecting changed modules since last build...'
                script {
                    def changedModules = []
                    def allModules = []
                    
                    // Get all Maven modules
                    def pomFiles = sh(
                        script: "find . -name 'pom.xml' -not -path './target/*' | head -20",
                        returnStdout: true
                    ).trim().split('\n')
                    
                    // Extract module names from pom files
                    for (pomFile in pomFiles) {
                        if (pomFile != './pom.xml') {
                            def moduleName = pomFile.replaceAll('./([^/]+)/pom.xml', '$1')
                            allModules.add(moduleName)
                        }
                    }
                    
                    // Get changed files since last successful build
                    def lastBuildCommit = ""
                    try {
                        // Try to get last successful build commit
                        def lastBuild = currentBuild.getPreviousSuccessfulBuild()
                        if (lastBuild) {
                            lastBuildCommit = sh(
                                script: "git log --format='%H' -n 1 --before='${lastBuild.getTime()}'",
                                returnStdout: true
                            ).trim()
                        }
                    } catch (Exception e) {
                        echo "No previous successful build found, will check last 5 commits"
                    }
                    
                    // If no previous build, check recent commits
                    if (!lastBuildCommit) {
                        lastBuildCommit = sh(
                            script: "git log --format='%H' -n 1 HEAD~5",
                            returnStdout: true
                        ).trim()
                    }
                    
                    // Get changed files
                    def changedFiles = ""
                    if (lastBuildCommit) {
                        changedFiles = sh(
                            script: "git diff --name-only ${lastBuildCommit} HEAD || git diff --name-only HEAD~1 HEAD",
                            returnStdout: true
                        ).trim()
                    } else {
                        changedFiles = sh(
                            script: "git diff --name-only HEAD~1 HEAD",
                            returnStdout: true
                        ).trim()
                    }
                    
                    echo "Changed files: ${changedFiles}"
                    
                    // Determine which modules are affected
                    if (changedFiles) {
                        def files = changedFiles.split('\n')
                        def affectedModules = [] as Set
                        
                        for (file in files) {
                            // Check if file belongs to a module
                            for (module in allModules) {
                                if (file.startsWith("${module}/") || 
                                    file.startsWith("${module}\\") ||
                                    file.contains("/${module}/") ||
                                    file.contains("\\${module}\\")) {
                                    affectedModules.add(module)
                                }
                            }
                            
                            // Also check for common files that affect all modules
                            if (file.equals('pom.xml') || 
                                file.startsWith('common/') ||
                                file.startsWith('shared/') ||
                                file.contains('parent') ||
                                file.contains('config')) {
                                echo "Core file changed: ${file} - will trigger full build"
                                affectedModules.addAll(allModules)
                                break
                            }
                        }
                        
                        changedModules = affectedModules.toList()
                    }
                    
                    // Force full build if requested or if no changes detected
                    if (params.FORCE_FULL_BUILD || changedModules.isEmpty()) {
                        echo "🔄 Full build requested or no specific changes detected"
                        changedModules = allModules
                    }
                    
                    echo "📋 Modules to build: ${changedModules.join(', ')}"
                    echo "📊 Total modules: ${allModules.size()}, Changed modules: ${changedModules.size()}"
                    
                    // Store in environment for later stages
                    env.CHANGED_MODULES = changedModules.join(',')
                    env.ALL_MODULES = allModules.join(',')
                    env.BUILD_TYPE = changedModules.size() == allModules.size() ? 'FULL' : 'INCREMENTAL'
                }
            }
        }
        
        stage('Production Build Info') {
            steps {
                echo "🏭 PRODUCTION Build Information:"
                echo "Environment: ${ENV_NAME}"
                echo "Branch/Tag: ${params.VERSION_TAG ?: params.RELEASE_BRANCH}"
                echo "Build number: ${env.BUILD_NUMBER}"
                echo "Build Type: ${env.BUILD_TYPE}"
                echo "Modules to build: ${env.CHANGED_MODULES}"
                echo "Create backup: ${params.CREATE_BACKUP}"
                echo "Deploy to prod: ${params.DEPLOY_TO_PROD}"
                sh 'java -version'
                sh 'mvn -version'
                sh 'git log --oneline -3'
            }
        }
        
        stage('Production Security Check') {
            steps {
                echo '🔐 Running production security validations...'
                script {
                    def allowedBranches = ['master','release/*', 'main']
                    def currentBranch = params.RELEASE_BRANCH
                    
                    if (params.VERSION_TAG) {
                        echo "✅ Building from version tag: ${params.VERSION_TAG}"
                    } else if (currentBranch.startsWith('release/') || allowedBranches.contains(currentBranch)) {
                        echo "✅ Building from approved branch: ${currentBranch}"
                    } else {
                        error("❌ Production builds only allowed from master, main, or release/* branches")
                    }
                }
            }
        }
        
        stage('Production Clean & Prepare') {
            steps {
                echo '🧹 Cleaning workspace for production build...'
                script {
                    if (env.BUILD_TYPE == 'FULL') {
                        sh 'mvn clean -q -T 2C'
                    } else {
                        // Clean only changed modules
                        def modules = env.CHANGED_MODULES.split(',')
                        for (module in modules) {
                            sh "mvn clean -q -f ${module}/pom.xml || true"
                        }
                    }
                }
            }
        }
        
        stage('Incremental Production Build') {
            steps {
                echo "📦 Building ${env.BUILD_TYPE} - ${env.CHANGED_MODULES.split(',').length} modules..."
                timeout(time: 60, unit: 'MINUTES') {
                    script {
                        if (env.BUILD_TYPE == 'FULL') {
                            // Full build
                            sh '''
                                echo "=== FULL PRODUCTION Build ==="
                                mvn package \
                                    -P${BUILD_PROFILE} \
                                    -DskipTests \
                                    -Dmaven.test.skip=true \
                                    -Dmaven.javadoc.skip=true \
                                    -Dmaven.source.skip=true \
                                    -Dcheckstyle.skip=false \
                                    -Dspotbugs.skip=false \
                                    -Denforcer.skip=false \
                                    -Dmaven.compiler.optimize=true \
                                    -Dmaven.compiler.debug=false \
                                    -T 2C
                            '''
                        } else {
                            // Incremental build
                            echo "=== INCREMENTAL PRODUCTION Build ==="
                            def modules = env.CHANGED_MODULES.split(',')
                            
                            // Build modules with dependencies
                            def moduleList = modules.join(',')
                            sh """
                                echo "Building modules: ${moduleList}"
                                
                                # Build changed modules and their dependents
                                mvn package \
                                    -P${BUILD_PROFILE} \
                                    -pl ${moduleList} \
                                    -am \
                                    -DskipTests \
                                    -Dmaven.test.skip=true \
                                    -Dmaven.javadoc.skip=true \
                                    -Dmaven.source.skip=true \
                                    -Dcheckstyle.skip=false \
                                    -Dspotbugs.skip=false \
                                    -Denforcer.skip=false \
                                    -Dmaven.compiler.optimize=true \
                                    -Dmaven.compiler.debug=false \
                                    -T 2C
                            """
                        }
                        
                        sh '''
                            echo "=== Build Process Completed ==="
                            echo "Packages created:"
                            find . -name "*.jar" -newer pom.xml 2>/dev/null | head -15
                        '''
                    }
                }
            }
        }
        
        stage('Production Package Validation') {
            steps {
                echo '✅ Validating built packages...'
                sh '''
                    echo "=== Production Package Validation ==="
                    
                    # Check if packages exist for changed modules
                    CHANGED_MODULES="${CHANGED_MODULES}"
                    if [ -n "$CHANGED_MODULES" ]; then
                        for module in ${CHANGED_MODULES//,/ }; do
                            echo "Checking module: $module"
                            MODULE_JARS=$(find "$module" -name "*.jar" 2>/dev/null | wc -l)
                            if [ "$MODULE_JARS" -gt 0 ]; then
                                echo "  ✅ $module: $MODULE_JARS packages found"
                                find "$module" -name "*.jar" -exec ls -lh {} \\; 2>/dev/null | head -3
                            else
                                echo "  ⚠️ $module: No packages found (might be parent module)"
                            fi
                        done
                    fi
                    
                    # Overall validation
                    TOTAL_JARS=$(find . -name "*.jar" -newer pom.xml 2>/dev/null | wc -l)
                    echo "Total packages created: $TOTAL_JARS"
                    
                    if [ "$TOTAL_JARS" -eq 0 ]; then
                        echo "❌ WARNING: No packages found!"
                        echo "This might be normal for parent modules or configuration changes"
                    else
                        echo "✅ Package validation completed"
                    fi
                '''
            }
        }
        
        stage('Create Production Backup') {
            when {
                expression { params.CREATE_BACKUP == true }
            }
            steps {
                echo '💾 Creating production backup...'
                sh '''
                    BACKUP_DIR="production-backup-${BUILD_NUMBER}-$(date +%Y%m%d_%H%M%S)"
                    mkdir -p "$BACKUP_DIR"
                    
                    # Copy packages for changed modules
                    if [ -n "${CHANGED_MODULES}" ]; then
                        for module in ${CHANGED_MODULES//,/ }; do
                            if [ -d "$module/target" ]; then
                                mkdir -p "$BACKUP_DIR/$module"
                                find "$module/target" -name "*.jar" -exec cp {} "$BACKUP_DIR/$module/" \\; 2>/dev/null || true
                            fi
                        done
                    fi
                    
                    # Create backup archive
                    if [ -n "$(ls -A $BACKUP_DIR 2>/dev/null)" ]; then
                        tar -czf "${BACKUP_DIR}.tar.gz" "$BACKUP_DIR"
                        echo "✅ Production backup created: ${BACKUP_DIR}.tar.gz"
                        ls -lh "${BACKUP_DIR}.tar.gz"
                    else
                        echo "ℹ️ No packages to backup"
                    fi
                    
                    rm -rf "$BACKUP_DIR"
                '''
                
                archiveArtifacts artifacts: 'production-backup-*.tar.gz', allowEmptyArchive: true
            }
        }
        
        stage('Archive Production Packages') {
            steps {
                echo '📁 Archiving production packages...'
                script {
                    def prodJars = sh(
                        script: "find . -name '*.jar' -newer pom.xml | grep -v test",
                        returnStdout: true
                    ).trim()
                    
                    if (prodJars) {
                        echo "Archiving ${env.BUILD_TYPE.toLowerCase()} build packages"
                        
                        // Archive all built packages
                        archiveArtifacts artifacts: '**/target/*.jar', 
                                       excludes: '**/target/*-tests.jar,**/target/*-sources.jar',
                                       allowEmptyArchive: true,
                                       fingerprint: true
                        
                        // Create a release package for changed modules
                        sh '''
                            RELEASE_DIR="thingsboard-${BUILD_TYPE,,}-${BUILD_NUMBER}"
                            mkdir -p "$RELEASE_DIR"
                            
                            # Copy main application JARs
                            find . -name "thingsboard*.jar" -o -name "application*.jar" | grep -v test | while read jar; do
                                cp "$jar" "$RELEASE_DIR/" 2>/dev/null || true
                            done
                            
                            # Create release info
                            echo "Build Type: ${BUILD_TYPE}" > "$RELEASE_DIR/build-info.txt"
                            echo "Modules: ${CHANGED_MODULES}" >> "$RELEASE_DIR/build-info.txt"
                            echo "Build: ${BUILD_NUMBER}" >> "$RELEASE_DIR/build-info.txt"
                            echo "Date: $(date)" >> "$RELEASE_DIR/build-info.txt"
                            
                            if [ -n "$(ls -A $RELEASE_DIR/*.jar 2>/dev/null)" ]; then
                                tar -czf "${RELEASE_DIR}.tar.gz" "$RELEASE_DIR"
                                echo "✅ Release package created: ${RELEASE_DIR}.tar.gz"
                            else
                                echo "ℹ️ No main application JARs to package"
                            fi
                            
                            rm -rf "$RELEASE_DIR"
                        '''
                        
                        archiveArtifacts artifacts: 'thingsboard-*-*.tar.gz', allowEmptyArchive: true
                    }
                }
            }
        }
        
        stage('Production Deployment Approval') {
            when {
                expression { params.DEPLOY_TO_PROD == true }
            }
            steps {
                script {
                    def approval = input(
                        message: "🚨 Deploy ${env.BUILD_TYPE} build to PRODUCTION?",
                        parameters: [
                            choice(name: 'DEPLOY_DECISION', 
                                   choices: ['ABORT', 'DEPLOY'], 
                                   description: 'Confirm production deployment')
                        ]
                    )
                    
                    if (approval == 'ABORT') {
                        error('Production deployment aborted by user')
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                expression { params.DEPLOY_TO_PROD == true }
            }
            steps {
                echo "🚀 Deploying ${env.BUILD_TYPE} build to PRODUCTION..."
                script {
                    sh '''
                        echo "=== PRODUCTION DEPLOYMENT ==="
                        echo "Target: ${DEPLOY_TARGET}"
                        echo "Build Type: ${BUILD_TYPE}"
                        echo "Modules: ${CHANGED_MODULES}"
                        echo "Build: ${BUILD_NUMBER}"
                        
                        # Deploy only changed modules (customize as needed)
                        if [ "${BUILD_TYPE}" = "INCREMENTAL" ]; then
                            echo "Deploying incremental changes..."
                            # Example: Deploy only changed services
                            # for module in ${CHANGED_MODULES//,/ }; do
                            #     echo "Deploying module: $module"
                            #     scp $module/target/*.jar prod-server:/opt/thingsboard/modules/$module/
                            # done
                        else
                            echo "Deploying full build..."
                            # Example: Deploy full release
                            # scp thingsboard-full-*.tar.gz prod-server:/opt/thingsboard/releases/
                        fi
                        
                        echo "✅ PRODUCTION deployment completed successfully"
                    '''
                }
            }
        }
        
        stage('Production Build Report') {
            steps {
                echo '📊 Generating production build report...'
                sh '''
                    echo "=== ThingsBoard PRODUCTION Build Report ===" > production-build-report.txt
                    echo "Environment: PRODUCTION" >> production-build-report.txt
                    echo "Build: ${BUILD_NUMBER}" >> production-build-report.txt
                    echo "Build Type: ${BUILD_TYPE}" >> production-build-report.txt
                    echo "Branch/Tag: ${VERSION_TAG:-$RELEASE_BRANCH}" >> production-build-report.txt
                    echo "Modules Built: ${CHANGED_MODULES}" >> production-build-report.txt
                    echo "Build Date: $(date)" >> production-build-report.txt
                    echo "Backup Created: ${CREATE_BACKUP}" >> production-build-report.txt
                    echo "Deployed: ${DEPLOY_TO_PROD}" >> production-build-report.txt
                    echo "" >> production-build-report.txt
                    
                    echo "=== Git Information ===" >> production-build-report.txt
                    git log --oneline -5 >> production-build-report.txt
                    echo "" >> production-build-report.txt
                    
                    echo "=== Packages Created ===" >> production-build-report.txt
                    find . -name "*.jar" -newer pom.xml -exec ls -lh {} \\; >> production-build-report.txt 2>/dev/null || echo "No packages found" >> production-build-report.txt
                    echo "" >> production-build-report.txt
                    
                    echo "=== Build Statistics ===" >> production-build-report.txt
                    echo "Total packages: $(find . -name "*.jar" -newer pom.xml | wc -l)" >> production-build-report.txt
                    echo "Build completed: $(date)" >> production-build-report.txt
                '''
                
                archiveArtifacts artifacts: 'production-build-report.txt', allowEmptyArchive: true
            }
        }
    }
    
    post {
        always {
            echo '🏁 Production pipeline completed!'
            sh 'rm -rf target/*/target || true'
        }
        success {
            echo "✅ ${env.BUILD_TYPE} production build succeeded!"
            script {
                def version = params.VERSION_TAG ?: params.RELEASE_BRANCH
                def moduleCount = env.CHANGED_MODULES.split(',').length
                currentBuild.description = "✅ ${env.BUILD_TYPE} build (${moduleCount} modules) - ${version}"
            }
        }
        failure {
            echo "❌ ${env.BUILD_TYPE} production build failed!"
            script {
                currentBuild.description = "❌ ${env.BUILD_TYPE} build failed at ${env.STAGE_NAME}"
                
                sh '''
                    echo "CRITICAL: Production build failure" > prod-failure.txt
                    echo "Build Type: ${BUILD_TYPE}" >> prod-failure.txt
                    echo "Modules: ${CHANGED_MODULES}" >> prod-failure.txt
                    echo "Stage: ${STAGE_NAME}" >> prod-failure.txt
                    echo "Build: ${BUILD_NUMBER}" >> prod-failure.txt
                    echo "Time: $(date)" >> prod-failure.txt
                '''
                archiveArtifacts artifacts: 'prod-failure.txt', allowEmptyArchive: true
            }
        }
    }
}
