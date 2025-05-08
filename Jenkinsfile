environment {
    SONARQUBE_SCANNER = 'SonarScanner'
    SONARQUBE_SERVER = 'SonarQubeServer'
    NODE_ENV = 'production'
    WEBHOOK_URL = 'https://cb4f-192-245-162-37.ngrok-free.app/webhook'
}

stages {
    stage('Checkout Code') {
        steps {
            git url: 'https://github.com/your-username/mern-bookstore-personal.git', branch: 'main'
        }
    }

    stage('Install Dependencies') {
        steps {
            dir('client') {
                sh 'npm install'
            }
            dir('server') {
                sh 'npm install'
            }
        }
    }

    stage('Lint') {
        steps {
            dir('client') {
                sh 'npx eslint .'
            }
            dir('server') {
                sh 'npx eslint .'
            }
        }
    }

    stage('Test') {
        steps {
            dir('server') {
                sh 'npm test'
            }
            dir('client') {
                sh 'npm test'
            }
        }
    }

    stage('SonarQube Analysis') {
        environment {
            scannerHome = tool "${SONARQUBE_SCANNER}"
        }
        steps {
            withSonarQubeEnv("${SONARQUBE_SERVER}") {
                dir('client') {
                    sh '''${scannerHome}/bin/sonar-scanner \
                      -Dsonar.projectKey=mern-bookstore-client \
                      -Dsonar.sources=. \
                      -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info'''
                }
                dir('server') {
                    sh '''${scannerHome}/bin/sonar-scanner \
                      -Dsonar.projectKey=mern-bookstore-server \
                      -Dsonar.sources=. \
                      -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info'''
                }
            }
        }
    }

    stage('Build Frontend') {
        steps {
            dir('client') {
                sh 'npm run build'
            }
        }
    }

    stage('Deploy (Optional)') {
        steps {
            echo 'Add deployment script here: Docker, SSH, SCP, etc.'
        }
    }
}

post {
    always {
        script {
            withCredentials([
                string(credentialsId: 'jenkins-username', variable: 'JENKINS_USERNAME'),
                string(credentialsId: 'api-token', variable: 'API_TOKEN'),
                string(credentialsId: 'secret-key', variable: 'SECRET_KEY'),
                string(credentialsId: 'iv-key', variable: 'IV_KEY')
            ]) {
                def getRawJson = { url ->
                    sh(script: "curl -s -u '$JENKINS_USERNAME:$API_TOKEN' '${url}'", returnStdout: true).trim()
                }

                def buildData = getRawJson("${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/api/json")
                def stageDescribe = getRawJson("${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/wfapi/describe")

                writeFile file: 'stageDescribe.json', text: stageDescribe
                def parsedDescribe = readJSON file: 'stageDescribe.json'

                def nodeStageDataStr = parsedDescribe.stages.collect { stage ->
                    def nodeId = stage.id
                    def nodeData = getRawJson("${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/execution/node/${nodeId}/wfapi/describe")
                    return """{"nodeId":${groovy.json.JsonOutput.toJson(nodeId)},"data":${nodeData}}"""
                }.join(',')

                def payloadStr = """{"build_data": ${buildData}, "node_stage_data": [${nodeStageDataStr}]}"""
                writeFile file: 'payload.json', text: payloadStr

                def checksum = sh(script: "sha256sum payload.json | awk '{print \$1}'", returnStdout: true).trim()

                def timestamp = System.currentTimeMillis().toString()
                def encryptedTimestamp = sh(script: """
                    echo -n '${timestamp}' | openssl enc -aes-256-cbc -base64 \
                    -K ${SECRET_KEY} \
                    -iv ${IV_KEY}
                """, returnStdout: true).trim()

                sh """
                    curl -X POST '${WEBHOOK_URL}' \
                    -H "Content-Type: application/json" \
                    -H "X-Encrypted-Timestamp: ${encryptedTimestamp}" \
                    -H "X-Checksum: ${checksum}" \
                    --data-binary @payload.json
                """
            }
        }
    }
}
