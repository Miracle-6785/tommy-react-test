pipeline {
    agent any

    tools {
        nodejs 'Node20'
    }

    environment {
        MARKER_FILE = ".jenkins_commit_marker_${env.BUILD_ID}"
        IS_DEPLOY_ON_S3 = "true"
        IS_USING_SONARQUBE = "true"
        IS_USING_DEPENDENCY_TRACK = "true"
        CLOUD_SERVICE_ANNOTATION = "cloud-service/app-info"
        DEPENDENCY_TRACK_ANNOTATION = "dependencytrack/project-id"
        APP_NAME = "tommy-react-test"
        BUCKET_NAME = "tommy-react-test-s3-bucket".toLowerCase()
        AWS_REGION = "ap-southeast-1"
        CLOUD_SERVICE_BACKEND_URL = "https://cloud-service.da-icy.social"
        CLOUD_SERVICE_PROJECT_ID = "1"
        API_URL = "${CLOUD_SERVICE_BACKEND_URL}/1/applications"
        SCANNER_HOME = tool 'SonarScanner'
        SONAR_PROJECT_KEY = "tommy-react-test"
        SONAR_BASE_URL = "https://sonar.da-icy.social"
        DEPENDENCE_BASE_URL = "https://dtrack-be.da-icy.social"
        BACKSTAGE_BE_BASE_URL = "https://backstage-be.da-icy.social"
        FILE_PATH = "catalog-info.yaml"
        
    }

    options {
        skipDefaultCheckout false
    }

    stages {
        stage('Check Branch') {
            when {
                expression { env.BRANCH_NAME != 'main' }
            }
            steps {
                echo "❌ Skipping execution. This branch is not main."
                script {
                    currentBuild.result = 'ABORTED'
                    error("Pipeline stopped: Not on main branch.")
                }
            }
        }

        stage('Check for Skip') {
            steps {
                scmSkip(deleteBuild: true, skipPattern:'.*\\[ci skip\\].*')
            }
        }

        stage('Install packages') {
            steps {
                script {
                    sh '''
                    npm install
                    echo "✅ Packages installed successfully"
                    '''
                }
            }
        }

        stage('Run test & generate coverage and SBOM & update Cloud service annotation'){
            steps {
                script {
                    parallel(
                        'Run Tests & Generate Coverage': {
                            if (env.IS_USING_SONARQUBE == 'true') {
                                sh '''
                                npm run test -- --coverage
                                echo "✅ Coverage test completed successfully"
                                '''
                            } else {
                                echo "🚀 Skipping Tests & Coverage since IS_USING_SONARQUBE is not true."
                            }
                        },

                        "Generate SBOM": {
                            if (env.IS_USING_DEPENDENCY_TRACK == 'true') {
                                sh '''
                                npx @cyclonedx/cyclonedx-npm --output-file bom.json
                                echo "✅ SBOM generated successfully"
                                '''
                            } else {
                                echo "🚀 Skipping SBOM generation since IS_USING_DEPENDENCY_TRACK is not true."
                            }
                        },

                        "Check & Update Cloud service application information": {
                            if (env.IS_DEPLOY_ON_S3 == 'true'){
                                if (checkAnnotationNotExist(CLOUD_SERVICE_ANNOTATION, FILE_PATH)) {
                                    echo "🔍 cloud-service/app-info is empty. Fetching application information..."

                                    def applicationId = fetchApplicationUUID()

                                    echo "Application UUID: ${applicationId}"

                                    if (applicationId && applicationId != "null") {
                                        echo "✅ Found Project UUID: ${applicationId}"

                                        updateCatalogInfo(CLOUD_SERVICE_ANNOTATION, "${CLOUD_SERVICE_PROJECT_ID}/${applicationId}")

                                        gitCommit("Updated cloud-service/app-info in catalog-info.yaml")

                                        createMakerFile()
                                    } else {
                                        error "❌ Failed to fetch Cloud service application information!"
                                    }
                                } else {
                                    echo "✅ Cloud service application information is exist. Skipping update."
                                }
                            } else {
                                echo "🚀 Skipping Cloud service application information update since IS_DEPLOY_ON_S3 is not true."
                            }
                        }
                    )
                }
            }
        }

        stage('Upload report to analytics server & update dependeny track annotation') {
            steps {
                script {
                    parallel(
                        'SonarQube Analysis': {
                            if (env.IS_USING_SONARQUBE == 'true') {
                                withSonarQubeEnv('SonarQube') {
                                    sh """
                                    ${SCANNER_HOME}/bin/sonar-scanner \
                                        -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                                        -Dsonar.sources=./src \
                                        -Dsonar.host.url=$SONAR_BASE_URL \
                                        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                                    """
                                }
                            } else {
                                echo "🚀 Skipping SonarQube Analysis since IS_USING_SONARQUBE is not true."
                            }
                        },

                        'dependencyTrackPublisher': {
                            if (env.IS_USING_DEPENDENCY_TRACK == 'true') {
                                withCredentials([string(credentialsId: 'DPCred', variable: 'API_KEY')]) {
                                    dependencyTrackPublisher artifact: 'bom.json', projectName: '$APP_NAME', projectVersion: '1.1.1', synchronous: true, dependencyTrackApiKey: API_KEY
                                }
                                sh 'echo "✅ Dependency Track scan completed successfully"'
                            } else {
                                echo "🚀 Skipping Dependency-Track Publisher since IS_USING_DEPENDENCY_TRACK is not true."
                            }
                        },

                        'Build React App': {
                            sh '''
                            npm run build
                            echo "✅ React app build completed successfully"
                            '''
                        },

                        'Check & Update Dependency-Track Project ID': {
                            if (env.IS_USING_DEPENDENCY_TRACK == 'true') {
                                withCredentials([string(credentialsId: 'DPCred', variable: 'API_KEY')]) {
                                    script {
                                        if (checkAnnotationNotExist(DEPENDENCY_TRACK_ANNOTATION, FILE_PATH)) {
                                            echo "🔍 dependencytrack/project-id is empty. Fetching Project UUID from Dependency-Track..."

                                            def projectId = fetchProjectUUID(APP_NAME, API_KEY)

                                            if (projectId && projectId != "null") {
                                                echo "✅ Found Project UUID: ${projectId}"

                                                updateCatalogInfo(DEPENDENCY_TRACK_ANNOTATION, projectId)

                                                gitCommit("Updated dependencytrack/project-id in catalog-info.yaml")

                                                createMakerFile()
                                            } else {
                                                error "❌ Failed to fetch Project UUID from Dependency-Track!"
                                            }
                                        } else {
                                            echo "✅ dependencytrack/project-id is already set. Skipping update."
                                        }
                                    }
                                }
                            } else {
                                echo "🚀 Skipping Dependency-Track Project ID update since IS_USING_DEPENDENCY_TRACK is not true."
                            }
                        }
                    )
                }
            }
        }

        stage('Test AWS S3 Connection') {
            when {
                expression { return env.IS_DEPLOY_ON_S3 == 'true' } // ✅ Only runs if IS_DEPLOY_ON_S3 is true
            }
            steps {
                script {
                    withAWS(credentials: 'miracle-aws-creds', region: AWS_REGION) {
                        def maxRetries = 15
                        def attempt = 0
                        def isRunning = false

                        while (attempt < maxRetries) {
                            def response = sh(script: "curl -s $API_URL | jq -r '.data[] | select(.name == \"$APP_NAME\") | {name: .name, status: .status}'", returnStdout: true).trim()

                            if (response) {
                                def appStatus = sh(script: "echo '$response' | jq -r '.status'", returnStdout: true).trim()
                                if (appStatus == "Running") {
                                    echo "✅ S3 bucket exists and is in RUNNING state: $BUCKET_NAME"
                                    isRunning = true
                                    break
                                } else {
                                    echo "⏳ S3 bucket exists but is NOT running (Status: $appStatus). Retrying..."
                                }
                            } else {
                                echo "🚀 S3 bucket does NOT exist. Creating a new bucket..."
                                isRunning = true
                                break
                            }
                            attempt++
                            sleep 10
                        }

                        if (!isRunning) {
                            error "❌ Timeout: S3 bucket is not in RUNNING state after ${maxRetries * 10} seconds. Exiting..."
                        }
                    }
                }
            }
        }

        stage('Configure & Deploy to S3') {
            when {
                expression { return env.IS_DEPLOY_ON_S3 == 'true' } // ✅ Only runs if IS_DEPLOY_ON_S3 is true
            }
            steps {
                script {
                    withAWS(credentials: 'miracle-aws-creds', region: AWS_REGION) {
                        sh """
                        aws s3 website s3://$BUCKET_NAME --index-document index.html --error-document index.html
                        aws s3 sync ./dist s3://$BUCKET_NAME --acl public-read
                        echo "✅ React app deployed to S3 bucket: $BUCKET_NAME"
                        """
                    }
                }
            }
        }

        stage('Push changes') {
            when {
                expression { return fileExists(env.MARKER_FILE) }
            }
            steps {
                script {
                    echo "Commit marker file found. Proceeding with push..."

                    pushChanges()

                    deregisterWithBackstage()

                    registerWithBackstage()

                    sh "rm -f ${env.MARKER_FILE}"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "🎉 Pipeline completed successfully!"
        }
    }
}

def createMakerFile() {
    sh "touch ${env.MARKER_FILE}"
    echo "Marker file ${env.MARKER_FILE} created."
}

def gitCommit(commitMessage) {
    withCredentials([usernamePassword(credentialsId: 'adminGithubCred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
        script {
            def repoUrl = sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
            repoUrl = repoUrl.startsWith("git@") ? repoUrl.replace("git@", "https://").replace(".com:", ".com/") : repoUrl

            echo "🔄 Using Git repository: ${repoUrl}"
            sh """
            git config --global user.email "miracleiztb@gmail.com"
            git config --global user.name "Miracle"
            git add ${FILE_PATH}
            git commit -m "[ci skip] ${commitMessage}"
            """
            echo "✅ Changes committed successfully!"
        }
    }
}

def pushChanges() {
    withCredentials([usernamePassword(credentialsId: 'adminGithubCred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
        script {
            def repoUrl = sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
            repoUrl = repoUrl.startsWith("git@") ? repoUrl.replace("git@", "https://").replace(".com:", ".com/") : repoUrl

            echo "🔄 Using Git repository: ${repoUrl}"
            sh """
            git push https://${GIT_USER}:${GIT_PASS}@${repoUrl.replace('https://', '')} HEAD:main
            """
            echo "✅ Changes committed and pushed successfully!"
        }
    }
}

def checkAnnotationNotExist(annotation, filePath) {
    def value = sh(script: "grep '${annotation}:' ${filePath} | awk -F': ' '{print \$2}'", returnStdout: true).trim()
    check = value.replaceAll('"', '')
    return check == "" || check == "null"
}

def fetchApplicationUUID() {
    def response = sh(script: """
        curl -s -X 'GET' '${CLOUD_SERVICE_BACKEND_URL}/${CLOUD_SERVICE_PROJECT_ID}/applications' \
        -H 'accept: application/json'
    """, returnStdout: true).trim()

    def applicationId = sh(script: """
        clean_res=\$(echo '${response}' | tr -d '\\000-\\031')
        echo "\$clean_res" | jq -r '.data[] | select(.name == "${APP_NAME}") | .uuid'
    """, returnStdout: true).trim()

    return applicationId
}

def fetchProjectUUID(appName, apiKey) {
    def response = sh(script: """
        curl -s -X 'GET' '${DEPENDENCE_BASE_URL}/api/v1/project/lookup?name=${appName}&version=1.1.1' \
        -H 'accept: application/json' \
        -H 'X-Api-Key: ${apiKey}'
    """, returnStdout: true).trim()

    def projectId = sh(script: "echo '${response}' | jq -r '.uuid'", returnStdout: true).trim()

    return projectId
}

def updateCatalogInfo(key, value) {
    def exists = sh(script: "grep -q '^[[:space:]]*${key}:' ${FILE_PATH}", returnStatus: true) == 0

    if (exists) {
        sh "sed -i 's|${key}: \"\"|${key}: \"${value}\"|' ${FILE_PATH}"
    } else {
        sh "sed -i '/annotations:/a\\    ${key}: \"${value}\"' ${FILE_PATH}"
    }
    echo "✅ ${key} updated successfully!"
}

def deregisterWithBackstage() {
    withCredentials([string(credentialsId: 'BackstageAPIKey', variable: 'API_KEY')]) {
        script {
            def repoUrl = sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
            def cleanedRepoUrl = repoUrl.replace('.git', '')

            def locations = sh(script: """
                curl -s -X GET "${BACKSTAGE_BE_BASE_URL}/api/catalog/locations" \\
                    -H "accept: application/json" \\
                    -H "Content-Type: application/json" \\
                    -H "Authorization: Bearer ${API_KEY}" \\
            """, returnStdout: true).trim()

            //  .[] | .data | select(.target == "https://github.com/Miracle-6785/demo25/tree/main/catalog-info.yaml") | .id

            def locationId = sh(script: """
                echo '${locations}' | jq -r '.[] | .data | select(.target == "${cleanedRepoUrl}/tree/main/catalog-info.yaml") | .id'
            """, returnStdout: true).trim()

            def res = sh(script: """
                curl -s -X DELETE "${BACKSTAGE_BE_BASE_URL}/api/catalog/locations/${locationId}" \\
                    -H "accept: application/json" \\
                    -H "Content-Type: application/json" \\
                    -H "Authorization: Bearer ${API_KEY}" \\
            """, returnStdout: true).trim()

            echo "✅ De-registered catalog-info.yaml with Backstage: ${locationId}"
        }
    }
}

def registerWithBackstage() {
    withCredentials([string(credentialsId: 'BackstageAPIKey', variable: 'API_KEY')]) {
        script {
            def repoUrl = sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
            def cleanedRepoUrl = repoUrl.replace('.git', '')

            def res = sh(script: """
                curl -s -X POST "${BACKSTAGE_BE_BASE_URL}/api/catalog/locations" \\
                    -H "accept: application/json" \\
                    -H "Content-Type: application/json" \\
                    -H "Authorization: Bearer ${API_KEY}" \\
                    -d '{
                        "type": "url",
                        "target": "${cleanedRepoUrl}/blob/main/catalog-info.yaml"
                    }'
            """, returnStdout: true).trim()

            echo "✅ Registered catalog-info.yaml with Backstage: ${res}"
        }
    }
}
