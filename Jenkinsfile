pipeline {
  environment {
        ARGO_SERVER = '34.205.144.35:32100'
        }
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    stage('Test') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        // stage('SCA') {
        //   steps {
        //     container('maven') {
        //       catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        //       sh 'mvn org.owasp:dependency-check-maven:check'
        //       }
        //     }
        //   }
        //   post {
        //     always {
        //       archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
        //       // dependencyCheckPublisher pattern: 'report.xml'
        //         }
        //       }
        //     }
        // stage('OSS License Checker') {
        //   steps {
        //     container('licensefinder') {
        //       sh 'ls -al'
        //       sh '''#!/bin/bash --login
        //             /bin/bash --login
        //             rvm use default
        //             gem install license_finder
        //             license_finder
        //             '''
        //                 }
        //               }
        //           }
        }
      }
    stage('Package') {
      parallel {
        // stage('Create Jarfile') {
        //   steps {
        //     container('maven') {
        //       sh 'mvn package -DskipTests'
        //     }
        //   }
        // }
        stage('OCI image build') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f "$(pwd)/Dockerfile-nonrootuser" -c "$(pwd)" --insecure --skip-tls-verify --cache=true --destination=docker.io/chandikas/dso-demo --verbosity=debug' 
              }
            }
          }
    }
    }
    stage('OCI Image Analysis') {
      parallel {
        stage('Image Linting') {
          steps {
            container('docker-tools') {
              sh 'dockle docker.io/chandikas/dso-demo'
                }
            }
          }
        stage('Image Scan') {
          steps {
            container('docker-tools') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                  sh 'trivy image --exit-code 1 docker.io/chandikas/dso-demo'
                      }              
                  }
              }
            }
          }
      }
    stage('Deploy to Dev') {
      environment {
        AUTH_TOKEN = credentials('argocd-deployer-token')
            }
        steps {
            container('docker-tools') {
                sh 'docker run -t schoolofdevops/argocd-cli argocd app sync devsecops --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
                sh 'docker run -t schoolofdevops/argocd-cli argocd app wait devsecops --health --timeout 300 --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
                }
            }
            }
    }
  }