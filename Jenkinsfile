pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  environment {
    APP_NAME = "sample-api-service"
    IMAGE_REGISTRY = "rmkanda"
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh './mvnw install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
        stage('Secrets Scanner') {
          steps {
            container('trufflehog') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE'){
                sh "trufflehog --exclude_paths secrets-exclude.txt ${GIT_URL}"
              }
            }
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh './mvnw package -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh './mvnw test'
            }
          }
        }
        stage('OSS check') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw org.owasp:dependency-check-maven:check'
              }
            }
          }  
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: false
              dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
            }
          }
        }
      }
    }
    stage('License-Finder') {
      steps {
        container('licensefinder') {
          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            sh '''#!/bin/bash --login
                    /bin/bash --login
                    rvm use default
                    gem install license_finder
                    license_finder
                  '''
          }
        }
      }
    }
    stage('OSS check') {
      steps {
        container('maven') {
          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            sh './mvnw compile spotbugs:check'
          }
        }
      }  
      post {
        always {
          recordIssues enableForFailure: true, tools: spotbugs()
        }
      }
    }
    stage('Package') {
      steps {
        container('docker-tools') {
          sh "docker build . -t ${APP_NAME}"
        }
      }
    }
    stage('Publish') {
      steps {
        container('docker-tools') {
          echo "Publishing docker image"
          // sh "docker push ${APP_NAME}"
        }
      }
    }
    stage('Docker image analysis') {
      parallel {
        stage('Container scan via grype') {
          steps {
            container('docker-tools') {
              sh 'grype ${APP_NAME} || exit 0'
            }
          }
        }
        stage('Container scan via dockle') {
          steps {
            container('docker-tools') {
              sh 'dockle ${APP_NAME}'
            }
          }  
        }
      }
    }
    stage('Deploy to Dev') {
      steps {
        container('docker-tools') {
          echo "Deploying the app"
          // sh "kubectl apply -f k8s.yaml"
        }
      }
    }
    stage('Promote to Prod') {
      steps {
        container('docker-tools') {
          echo "Promote to Prod"
        }
      }
    }
  }
}