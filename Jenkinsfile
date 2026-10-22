```groovy
pipeline {
    agent any

    parameters {
        choice(
            name: 'PIPELINE_TYPE',
            choices: ['parallel', 'conditional'],
            description: 'Choose the pipeline execution type'
        )

        choice(
            name: 'RELEASE_MODE',
            choices: ['stable', 'faulty'],
            description: 'Choose stable release or intentionally faulty release for rollback testing'
        )
    }

    environment {
        DOCKER_IMAGE = 'kushmithaa/transport-monitoring-dashboard'
        APP_NAME = 'transport-monitoring-dashboard'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Parallel Validation') {
            when {
                expression {
                    params.PIPELINE_TYPE == 'parallel'
                }
            }

            parallel {
                stage('HTML Check') {
                    steps {
                        bat '''
                        if not exist index.html exit /b 1
                        echo HTML file exists
                        '''
                    }
                }

                stage('Manifest Check') {
                    steps {
                        bat '''
                        if not exist deployment.yaml exit /b 1
                        echo Kubernetes manifest exists
                        '''
                    }
                }
            }
        }

        stage('Conditional Validation') {
            when {
                expression {
                    params.PIPELINE_TYPE == 'conditional'
                }
            }

            steps {
                bat '''
                if not exist index.html exit /b 1
                if not exist Dockerfile exit /b 1
                if not exist deployment.yaml exit /b 1
                echo All required files exist
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${env.BUILD_NUMBER}"

                    if (params.RELEASE_MODE == 'faulty') {
                        imageTag = "faulty-${env.BUILD_NUMBER}"
                    }

                    bat "docker build -t ${DOCKER_IMAGE}:${imageTag} ."
                    bat "docker tag ${DOCKER_IMAGE}:${imageTag} ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Local Smoke Test') {
            steps {
                bat '''
                docker rm -f transport-dashboard-test 2>NUL
                docker run -d --name transport-dashboard-test -p 18086:80 %DOCKER_IMAGE%:%BUILD_NUMBER%
                timeout /t 3 /nobreak >NUL
                curl.exe --fail http://localhost:18086/
                if errorlevel 1 (
                    docker logs transport-dashboard-test
                    docker rm -f transport-dashboard-test
                    exit /b 1
                )
                docker rm -f transport-dashboard-test
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    powershell '''
                    $ErrorActionPreference = "Stop"
                    $env:DOCKER_PASS | docker login -u $env:DOCKER_USER --password-stdin
                    if ($LASTEXITCODE -ne 0) {
                        exit $LASTEXITCODE
                    }
                    '''

                    bat "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                    bat "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    script {
                        if (params.RELEASE_MODE == 'faulty') {
                            echo 'Testing rollback: intentionally deploy a nonexistent image tag.'
                            bat '''
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/transport-monitoring-dashboard transport-monitoring-dashboard=%DOCKER_IMAGE%:nonexistent-faulty-tag
                            '''
                        } else {
                            bat '''
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" apply -f deployment.yaml
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/transport-monitoring-dashboard transport-monitoring-dashboard=%DOCKER_IMAGE%:%BUILD_NUMBER%
                            '''
                        }
                    }
                }
            }
        }

        stage('Verify Rollout and Roll Back if Needed') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    powershell '''
                    $ErrorActionPreference = "Continue"

                    kubectl --kubeconfig="$env:KUBECONFIG_FILE" rollout status deployment/transport-monitoring-dashboard --timeout=90s

                    if ($LASTEXITCODE -ne 0) {
                        Write-Host "Rollout failed. Rolling back to the previous deployment revision..."
                        kubectl --kubeconfig="$env:KUBECONFIG_FILE" rollout undo deployment/transport-monitoring-dashboard

                        if ($LASTEXITCODE -ne 0) {
                            Write-Error "Rollback command failed."
                            exit 1
                        }

                        kubectl --kubeconfig="$env:KUBECONFIG_FILE" rollout status deployment/transport-monitoring-dashboard --timeout=90s

                        if ($LASTEXITCODE -ne 0) {
                            Write-Error "Rollback did not restore a healthy rollout."
                            exit 1
                        }

                        Write-Host "Rollback successful. Previous deployment is healthy."
                    } else {
                        Write-Host "Deployment rollout successful."
                    }
                    '''
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile', fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review Console Output to identify the failed stage.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
```