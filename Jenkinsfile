pipeline {
    agent {
        docker {
            image 'node:18'
            label 'CentOS-32'
        }
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        WORKDIR = 'wdio_qei/wdio'
    }

    stages {

        stage('Checkout Repo') {
            steps {
                sh '''
                  if [ ! -d "$WORKDIR" ]; then
                    git clone https://github.com/qdbrowserstack/wdio_qei.git $WORKDIR
                  fi
                  cd $WORKDIR
                  git checkout main
                  git pull origin main
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                  cd $WORKDIR
                  node -v
                  npm -v

                  rm -rf node_modules package-lock.json .npm
                  export NPM_CONFIG_CACHE=$PWD/.npm
                  npm install
                '''
            }
        }

        stage('Decrypt Configs') {
            steps {
                withCredentials([
                    string(credentialsId: 'qei_encryption_key', variable: 'ENCRYPTION_KEY')
                ]) {
                    sh '''
                      cd $WORKDIR
                      export ENCRYPTION_KEY=$ENCRYPTION_KEY
                      npm run decrypt
                    '''
                }
            }
        }

        stage('Run WDIO Tests – Stage 1') {
            steps {
                script {
                    catchError(buildResult: 'UNSTABLE') {
                        withCredentials([
                            string(credentialsId: 'qei_encryption_key', variable: 'ENCRYPTION_KEY')
                        ]) {
                            sh '''
                              cd $WORKDIR
                              export ENCRYPTION_KEY=$ENCRYPTION_KEY
                              npm run test:stage1
                            '''
                        }
                    }
                }
            }
        }

        stage('Run WDIO Tests – Stage 2 (Failed Only)') {
            steps {
                script {
                    catchError(buildResult: 'UNSTABLE') {
                        withCredentials([
                            string(credentialsId: 'qei_encryption_key', variable: 'ENCRYPTION_KEY')
                        ]) {
                            sh '''
                              cd $WORKDIR
                              export ENCRYPTION_KEY=$ENCRYPTION_KEY
                              npm run test:stage2
                            '''
                        }
                    }
                }
            }
        }
    }

  post {
      always {
          script {
              if (fileExists('reports/junit')) {
                  junit testResults: '**/reports/junit/**/*.xml',
                        allowEmptyResults: true

                  archiveArtifacts artifacts: '**/reports/junit/**/*.xml',
                                  allowEmptyArchive: true,
                                  fingerprint: true
              } else {
                  echo 'No JUnit reports found, skipping publish step'
              }
          }
      }
  }
}
