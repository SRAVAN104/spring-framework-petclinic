pipeline {
    agent any
    tools {
        maven 'maven' 
    }
    environment {
        SONARQUBE_SERVER = 'sonar' 
        DOCKERHUB_REPO = 'sravan614/petclinic'
        COMMIT_ID = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
    }
    parameters {
        choice(name: 'ZAP_SCAN_TYPE', choices: ['Baseline', 'API', 'FULL'], description: 'Choose the type of OWASP ZAP scan to run')
    }
    stages {
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
                    //def imageTag = "${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"
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
                    sh "docker build -t ${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID} ."
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
                }
            }
        }

        

        stage('Trivy Image Scan') {
            steps {
                script {
                    echo "Scanning Docker Image with Trivy"
                    sh 'trivy image --download-db-only'
                    //def imageTag = "${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL --format json -o trivy_report.json ${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"

                    archiveArtifacts artifacts: 'trivy_report.json', allowEmptyArchive: true
                }
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    script {
                      /*  sh '''
                            sudo apt-get install -y python3
                            python3 --version
                        '''*/
                        echo "Selected OWASP ZAP scan type: ${params.ZAP_SCAN_TYPE}"
                        def zapScript = params.ZAP_SCAN_TYPE == 'Baseline' ? 'zap-baseline.py' : (params.ZAP_SCAN_TYPE == 'API' ? 'zap-api-scan.py' : 'zap-full-scan.py')
                        def status = sh(script: '''
                            docker run -v $PWD:/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable python3 /zap/${zapScript} -t http://54.89.252.41:4000/ > ${zapScript}.html
                        ''', returnStatus: true)

                        if (status == 0) {
                            echo "ZAP scan completed successfully."
                        } else {
                            error "ZAP scan failed with status code: ${status}"
                        }
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo "Pushing Docker Image to DockerHub"
                  //  def imageTag = "${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
                        sh "docker push ${DOCKERHUB_REPO}:${env.BUILD_NUMBER}-${COMMIT_ID}"
                    }
                }
            }
        }
    }
    // post {
    //     always {
    //         echo "Cleaning up Docker resources"
    //         sh 'docker system prune -f'
    //     }
    //     success {
    //         mail to: 'lathasree.chillakuru@gmail.com',
    //              subject: "Jenkins Job - SUCCESS",
    //              body: "The Jenkins job has completed successfully."
    //     }
    //     failure {
    //         mail to: 'lathasree.chillakuru@gmail.com',
    //              subject: "Jenkins Job - FAILURE",
    //              body: "The Jenkins job has failed. Please review the logs."
    //     }
    // }
}
