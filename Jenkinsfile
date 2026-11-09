pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION       = 'ap-south-1'
        AWS_ACCOUNT_ID   = '699038657587'
        ECR_REPOSITORY   = 'devsecops-app'
        EKS_CLUSTER      = 'devsecops-eks'
        KUBECONFIG       = '/var/lib/jenkins/.kube/config'
        SONAR_PROJECT    = 'digital-marketing'
        APP_DIR          = '2128_tween_agency'
        DOCKER_IMAGE     = 'digital-marketing'
    }

    tools {
        sonarQube 'SonarScanner'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    echo "Repository checked out"
                    git branch --show-current
                    ls -la
                    test -f "${APP_DIR}/Dockerfile"
                    test -f "${APP_DIR}/sonar-project.properties"
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir("${APP_DIR}") {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT} \
                              -Dsonar.projectName="Digital Marketing" \
                              -Dsonar.sources=. \
                              -Dsonar.sourceEncoding=UTF-8
                        '''
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan 2128_tween_agency --format HTML --format XML --out dependency-check-report',
                    odcInstallation: 'DependencyCheck'
                )
            }
            post {
                always {
                    dependencyCheckPublisher(
                        pattern: 'dependency-check-report/dependency-check-report.xml',
                        failedTotalCritical: 1
                    )
                    archiveArtifacts(
                        artifacts: 'dependency-check-report/**',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                    env.ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
                    env.FULL_IMAGE = "${ECR_URI}:${IMAGE_TAG}"
                }

                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                      ${APP_DIR}

                    docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${FULL_IMAGE}
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                      --no-progress \
                      --ignore-unfixed \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      ${FULL_IMAGE}
                '''
            }
        }

        stage('Push to Amazon ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} \
                      | docker login \
                          --username AWS \
                          --password-stdin \
                          ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    docker push ${FULL_IMAGE}
                '''
            }
        }

        stage('Deploy to Amazon EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER} \
                      --kubeconfig ${KUBECONFIG}

                    kubectl --kubeconfig=${KUBECONFIG} \
                      apply -f /root/devsecops-eks-project/k8s/

                    kubectl --kubeconfig=${KUBECONFIG} \
                      set image deployment/digital-marketing \
                      digital-marketing=${FULL_IMAGE}

                    kubectl --kubeconfig=${KUBECONFIG} \
                      rollout status deployment/digital-marketing \
                      --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl --kubeconfig=${KUBECONFIG} \
                      get pods -l app=digital-marketing -o wide

                    kubectl --kubeconfig=${KUBECONFIG} \
                      get service digital-marketing-service
                '''
            }
        }
    }

    post {
        success {
            echo 'DevSecOps pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Review the stage logs before retrying.'
        }
        always {
            sh '''
                docker image prune -f || true
            '''
        }
    }
}
