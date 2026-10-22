pipeline {
  agent any
  parameters {
    choice(name: 'PIPELINE_TYPE', choices: ['parallel', 'conditional'], description: 'Pipeline concept to demonstrate')
    choice(name: 'RELEASE_MODE', choices: ['stable', 'faulty'], description: 'Faulty release is for rollback demonstration')
  }
  environment {
    IMAGE = 'kushmithaa/transport-monitoring-dashboard'
    DEPLOYMENT = 'transport-dashboard'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm; bat 'git log -1 --oneline' }
    }
    stage('Parallel Validation') {
      when { expression { params.PIPELINE_TYPE == 'parallel' } }
      parallel {
        stage('HTML check') {
          steps {
            powershell '''
              if (!(Test-Path "index.html")) { throw "index.html missing" }
              $h = Get-Content index.html -Raw
              if ($h -notmatch "<html" -or $h -notmatch "Public Transport Monitoring") { throw "HTML validation failed" }
              Write-Host "HTML validation passed"
            '''
          }
        }
        stage('Manifest check') {
          steps {
            powershell '''
              $y = Get-Content deployment.yaml -Raw
              if ($y -notmatch "kind: Deployment" -or $y -notmatch "kind: Service" -or $y -notmatch "replicas: 2") { throw "Manifest validation failed" }
              Write-Host "Kubernetes manifest validation passed"
            '''
          }
        }
      }
    }
    stage('Conditional Validation') {
      when { expression { params.PIPELINE_TYPE == 'conditional' } }
      steps {
        powershell '''
          $h = Get-Content index.html -Raw
          if ($h -notmatch "Passenger advisories") { throw "Passenger advisory section missing" }
          Write-Host "Conditional advisory check passed"
        '''
      }
    }
    stage('Build Image') {
      steps {
        script {
          if (params.RELEASE_MODE == 'faulty') {
            bat 'docker build -t %IMAGE%:faulty-%BUILD_NUMBER% .'
          } else {
            bat 'docker build -t %IMAGE%:build-%BUILD_NUMBER% .'
          }
        }
      }
    }
    stage('Local Smoke Test') {
      when { expression { params.RELEASE_MODE == 'stable' } }
      steps {
        bat 'docker run -d --rm --name transport-smoke-%BUILD_NUMBER% -p 18080:80 %IMAGE%:build-%BUILD_NUMBER%'
        powershell '''
          try {
            $r = Invoke-WebRequest http://localhost:18080 -UseBasicParsing -TimeoutSec 15
            if ($r.StatusCode -ne 200 -or $r.Content -notmatch "Public Transport Monitoring") { throw "Smoke test failed" }
          } finally { docker stop transport-smoke-$env:BUILD_NUMBER 2>$null }
        '''
      }
    }
    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          bat 'echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin'
          script {
            if (params.RELEASE_MODE == 'faulty') {
              bat 'docker push %IMAGE%:faulty-%BUILD_NUMBER%'
            } else {
              bat 'docker tag %IMAGE%:build-%BUILD_NUMBER% %IMAGE%:stable'
              bat 'docker push %IMAGE%:build-%BUILD_NUMBER%'
              bat 'docker push %IMAGE%:stable'
            }
          }
        }
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          script {
            if (params.RELEASE_MODE == 'faulty') {
              bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/%DEPLOYMENT% transport-dashboard=%IMAGE%:faulty-%BUILD_NUMBER%'
              bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%DEPLOYMENT% --timeout=45s'
            } else {
              bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/%DEPLOYMENT% transport-dashboard=%IMAGE%:build-%BUILD_NUMBER%'
              bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/%DEPLOYMENT% --timeout=120s'
              bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" get pods -l app=transport-dashboard'
            }
          }
        }
      }
      post {
        failure {
          script {
            if (params.RELEASE_MODE == 'faulty') {
              withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout undo deployment/transport-dashboard'
                bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/transport-dashboard --timeout=120s'
                bat 'kubectl --kubeconfig="%KUBECONFIG_FILE%" get pods -l app=transport-dashboard'
              }
              echo 'Failed rollout detected; previous ReplicaSet restored.'
            }
          }
        }
      }
    }
    stage('Archive Evidence') {
      steps { archiveArtifacts artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile', fingerprint: true }
    }
  }
  post { always { echo 'Review Console Output and Kubernetes rollout status.' } }
}
