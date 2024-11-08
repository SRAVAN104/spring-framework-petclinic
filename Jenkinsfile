pipeline {
    agent any
    tools {
        maven 'maven' 
    }
    environment {
        SONARQUBE_SERVER = 'SONARQUBE_SERVER' 
        DOCKERHUB_REPO = 'sravan614/petclinic'
        COMMIT_ID = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        ECR_URI = '241533123721.dkr.ecr.us-east-1.amazonaws.com/petclinic'  // Replace with your actual ECR URI
        AWS_REGION = 'us-east-1'  // Replace with your AWS region
        SLACK_CHANNEL = '#cloud'
        EMAIL_RECIPIENTS = 'thanatisravankumar2003@gmail.com'
    }
    parameters {
        choice(name: 'ZAP_SCAN_TYPE', choices: ['Baseline', 'API', 'FULL'], description: 'Choose the type of OWASP ZAP scan to run')
    }
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }
        stage('Unit Test') {
            steps {
                script {
                    echo "Running Maven Unit Tests"
                    sh 'mvn clean test'
                }
            }
        }
        stage('SAST with SonarQube') {
            steps {
                script {
                    echo "Running SonarQube SAST Scan"
                    withSonarQubeEnv(SONARQUBE_SERVER) {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                script {
                    timeout(time: 60, unit: 'MINUTES') {
                        dependencyCheck additionalArguments: '--scan ./ --disableNodeJS', odcInstallation: 'owasp'
                        dependencyCheckPublisher(
                            failedTotalCritical: 1,
                            failedTotalHigh: 5,
                            pattern: 'dependency-check-report.xml',
                            stopBuild: true,
                            unstableTotalCritical: 2,
                            unstableTotalHigh: 10
                        )
                    }
                }
            }
        }
        stage('Build WAR Package') {
            steps {
                script {
                    echo "Building WAR Package"
                    sh 'mvn clean package'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image'
                    def imageTag = "${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"
                    sh '''
                        cat <<EOF > Dockerfile
                        FROM tomcat:9.0-jdk17
                        RUN groupadd -r appgroup && useradd -r -g appgroup -m -d /app appuser
                        RUN chown -R appuser:appgroup /usr/local/tomcat
                        WORKDIR /app
                        COPY .mvn/ .mvn
                        COPY mvnw pom.xml ./ 
                        RUN chmod +x mvnw
                        USER appuser
                        COPY src ./src
                        EXPOSE 8080
                        CMD ["./mvnw", "jetty:run-war"]
                        
                    '''
                    // sh "docker build -t ${imageTag} ."
                }
            }
        }
        stage('Lint Dockerfile') {
            steps {
                script {
                    echo "Linting Dockerfile with Hadolint"
                    sh '''
                        docker run --rm -i hadolint/hadolint < Dockerfile > hadolint_report.txt
                    '''
                    archiveArtifacts artifacts: 'hadolint_report.txt', allowEmptyArchive: true
                    sh "docker build -t ${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID} ."
                }
            }
        }
       stage('Trivy Image Scan') {
    steps {
        script {
            echo "Scanning Docker Image with Trivy"
            def imageTag = "${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"
            def maxRetries = 3
            def retries = 0
            def scanSuccessful = false

            while (retries < maxRetries && !scanSuccessful) {
                try {
                    echo "Attempt ${retries + 1} of Trivy scan"
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL --format json --scanners vuln -o trivy_report.json $imageTag"
                    scanSuccessful = true
                } catch (Exception e) {
                    retries++
                    echo "Trivy scan attempt ${retries} failed. Retrying..."
                    sleep time: 10, unit: 'SECONDS'  // Pause briefly before retrying
                }
            }

            if (!scanSuccessful) {
                error "Trivy scan failed after ${maxRetries} attempts"
            }
            archiveArtifacts artifacts: 'trivy_report.json', allowEmptyArchive: true
        }
    }
}

stage('OWASP ZAP Scan') {
    steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
            script {
                echo "Selected OWASP ZAP scan type: ${params.ZAP_SCAN_TYPE}"
                def zapScript = ''
                def filetype = ''
                if (params.ZAP_SCAN_TYPE == 'Baseline') {
                    zapScript = 'zap-baseline.py'
                    filetype = 'zap-baseline.html'
                } else if (params.ZAP_SCAN_TYPE == 'API') {
                    zapScript = 'zap-api-scan.py'
                    filetype = 'zap-api-scan.html'
                } else if (params.ZAP_SCAN_TYPE == 'FULL') {
                    zapScript = 'zap-full-scan.py'
                    filetype = 'zap-full-scan.html'
                }

                // Run the ZAP scan and handle non-zero exit codes
                def status = sh(script: """
                    docker run -v $PWD:/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable python3 /zap/${zapScript} -t http://54.152.246.198:4000/ > ${filetype}
                """, returnStatus: true)

                env.FILE_TYPE = filetype
                echo "ZAP scan completed with status code: ${status}"

                if (status != 0) {
                    echo "ZAP scan found issues or encountered warnings (status code ${status})"
                    currentBuild.result = 'UNSTABLE'  // Mark build as unstable
                }
            }
        }
    }
}

        stage('Push to ECR') {
            steps {
                script {
                    echo "Logging into AWS ECR"
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URI}"
                    echo "Tagging Docker image"
                    def imageTag = "${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"
                    def ecrImageTag = "${ECR_URI}:${env.BUILD_NUMBER}-${COMMIT_ID}"
                    sh "docker tag ${imageTag} ${ecrImageTag}"
                    echo "Pushing image to ECR"
                    sh "docker push ${ecrImageTag}"
                }
            }
        }
        
    }
    post {
    always {
        script {
            withCredentials([string(credentialsId: 'SlackToken', variable: 'SLACK_TOKEN')]) {
                def message = "Jenkins Job - ${currentBuild.currentResult ?: 'SUCCESS'}: Build #${env.BUILD_NUMBER} in job '${env.JOB_NAME}' completed."
                slackSend(
                    channel: "${SLACK_CHANNEL}",
                    color: (currentBuild.result == 'SUCCESS') ? 'good' : 'warning',
                    message: message,
                    tokenCredentialId: 'SlackToken'
                )
            }
        }

        emailext (
            subject: "Jenkins Build #${env.BUILD_NUMBER}: ${currentBuild.result}",
            body: """
                Commit ID: ${COMMIT_ID}
                Build Link: ${env.BUILD_URL}
                Triggered By: ${env.BUILD_USER}

                Reports:
                - Trivy Report: ${env.WORKSPACE}/trivy_report.json
                - Hadolint Report: ${env.WORKSPACE}/hadolint_report.txt
                - OWASP ZAP Report: ${env.WORKSPACE}/${env.FILE_TYPE}
            """,
            to: "${EMAIL_RECIPIENTS}",
            attachmentsPattern: "trivy_report.json, hadolint_report.txt, ${env.FILE_TYPE}",  // Attach the reports
            attachLog: true  // Attach the build log as well for better debugging
        )
        echo "Cleaning up Docker resources"
        deleteDir()
    }
}
}
