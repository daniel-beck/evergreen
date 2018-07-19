pipeline {
    agent { label 'linux' }

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(daysToKeepStr: '10'))
        timestamps()
    }

    triggers {
        pollSCM('H * * * *')
        cron('H H * * *')
    }

    environment {
        // Squid is enabled by default in the dev environment
        DISABLE_PROXY_CACHE = 'true'
    }

    stages {
        stage('Prepare Workspace') {
            steps {
                sh 'make clean || true'
            }
        }

        stage('Lint code') {
          steps {
              githubNotify context: 'lint', description: 'make lint', status: 'PENDING'
              sh 'make lint'
          }
          post {
              success  { githubNotify context: 'lint', description: 'make lint', status: 'SUCCESS' }
              failure  { githubNotify context: 'lint', description: 'make lint', status: 'FAILURE' }
              unstable { githubNotify context: 'lint', description: 'make lint', status: 'FAILURE' }
          }
        }

        stage('Verifications') {
            post {
                success  { githubNotify context: 'verifications', description: 'NodeJS Checks', status: 'SUCCESS' }
                failure  { githubNotify context: 'verifications', description: 'NodeJS Checks', status: 'FAILURE' }
                unstable { githubNotify context: 'verifications', description: 'NodeJS Checks', status: 'FAILURE' }
            }
            parallel {
                stage('Evergreen Client') {
                    steps {
                        // notification not purely related to here, but there's no proper way with
                        // Declarative to run something before all parallel stages.
                        githubNotify context: 'verifications', description: 'NodeJS Checks', status: 'PENDING'
                        sh 'make -C distribution/client check'
                    }
                    post {
                        success {
                            archiveArtifacts 'distribution/client/coverage/**'
                        }
                    }
                }
                stage('Backend Services') {
                    steps {
                        sh 'make -C services check'
                    }
                    post {
                        success {
                            archiveArtifacts 'services/coverage/**'
                        }
                        cleanup {
                            sh 'make -C services stop'
                        }
                    }
                }
            }
        }

        stage('Build images') {
            post {
                success  { githubNotify context: 'build-images', description: 'Build Docker images', status: 'SUCCESS' }
                failure  { githubNotify context: 'build-images', description: 'Build Docker images', status: 'FAILURE' }
                unstable { githubNotify context: 'build-images', description: 'Build Docker images', status: 'FAILURE' }
            }
            parallel {

                stage('jenkins/evergreen') {
                    environment {
                        // Since tests have already been successfully run, skip them
                        SKIP_TESTS = 'true'
                    }
                    steps {
                        githubNotify context: 'build-images', description: 'Build Docker images', status: 'PENDING'
                        sh 'make -C distribution container'
                    }
                }

                stage('jenkinsciinfra/evergreen-backend') {
                    environment {
                        // Since tests have already been successfully run, skip them
                        SKIP_TESTS = 'true'
                    }
                    steps {
                        sh 'make -C services container'
                    }
                }
            }
        }

        stage('Test images') {
            parallel {
                stage('Base image') {
                  agent { label 'linux' }
                  steps {
                      githubNotify context: 'base-container-check', description: 'base-container-check', status: 'PENDING'
                      sh 'make -C distribution base-container-check'
                  }
                  post {
                      always {
                          archiveArtifacts artifacts: '**/build/tests-run*/**.log*'
                      }
                      success  { githubNotify context: 'base-container-check', description: 'base-container-check', status: 'SUCCESS' }
                      failure  { githubNotify context: 'base-container-check', description: 'base-container-check', status: 'FAILURE' }
                      unstable { githubNotify context: 'base-container-check', description: 'base-container-check', status: 'FAILURE' }
                  }
                }
                stage('Docker Cloud image') {
                  agent { label 'linux' }
                  steps {
                      githubNotify context: 'docker-cloud-container-check', description: 'docker-cloud-container-check', status: 'PENDING'
                      sh 'make -C distribution docker-cloud-container-check'
                  }
                  post {
                      always {
                          archiveArtifacts artifacts: '**/build/tests-run*/**.log*'
                      }
                      success  { githubNotify context: 'docker-cloud-container-check', description: 'docker-cloud-container-check', status: 'SUCCESS' }
                      failure  { githubNotify context: 'docker-cloud-container-check', description: 'docker-cloud-container-check', status: 'FAILURE' }
                      unstable { githubNotify context: 'docker-cloud-container-check', description: 'docker-cloud-container-check', status: 'FAILURE' }
                  }
                }
                stage('AWS Cloud image (smokes)') {
                  agent { label 'linux' }
                  steps {
                      githubNotify context: 'aws-cloud-container-check', description: 'aws-cloud-container-check', status: 'PENDING'
                      sh 'make -C distribution aws-cloud-container-check'
                  }
                  post {
                      always {
                          archiveArtifacts artifacts: '**/build/tests-run*/**.log*'
                      }
                      success  { githubNotify context: 'aws-cloud-container-check', description: 'aws-cloud-container-check', status: 'SUCCESS' }
                      failure  { githubNotify context: 'aws-cloud-container-check', description: 'aws-cloud-container-check', status: 'FAILURE' }
                      unstable { githubNotify context: 'aws-cloud-container-check', description: 'aws-cloud-container-check', status: 'FAILURE' }
                  }
                }
            }
        }

        stage('Publish jenkins/evergreen') {
            when {
                expression { infra.isTrusted() }
            }

            steps {
                withCredentials([[$class: 'ZipFileBinding',
                           credentialsId: 'jenkins-dockerhub',
                                variable: 'DOCKER_CONFIG']]) {
                    sh 'make publish'
                }
            }
        }

    }
}

// vim: ft=groovy
