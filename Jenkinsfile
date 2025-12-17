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
        REPO_DIR = 'wdio_qei/wdio'
        WORKDIR  = 'wdio_qei/wdio/wdio'
    }

    stages {

        stage('Checkout Repo') {
            steps {
                dir('wdio_qei') {
                    git branch: 'main',
                        url: 'https://github.com/qdbrowserstack/wdio_qei.git'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir("${WORKDIR}") {
                    sh '''
                      node -v
                      npm -v

                      rm -rf node_modules package-lock.json .npm
                      export NPM_CONFIG_CACHE=$PWD/.npm
                      npm install
                    '''
                }
            }
        }

        stage('Decrypt Configs') {
            steps {
                withCredentials([
                    string(credentialsId: 'qei_encryption_key', variable: 'ENCRYPTION_KEY')
                ]) {
                    dir("${WORKDIR}") {
                        sh '''
                          export ENCRYPTION_KEY=$ENCRYPTION_KEY
                          npm run decrypt
                        '''
                    }
                }
            }
        }

        stage('Run WDIO Tests – Stage 1') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    withCredentials([
                        string(credentialsId: 'qei_encryption_key', variable: 'ENCRYPTION_KEY')
                    ]) {
                        dir("${WORKDIR}") {
                            sh '''
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
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    withCredentials([
                        string(credentialsId: 'qei_encryption_key', variable: 'ENCRYPTION_KEY')
                    ]) {
                        dir("${WORKDIR}") {
                            sh '''
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
            echo "Publishing WDIO test results"
    
            junit(
                testResults: 'wdio_qei/wdio/reports/junit/*.xml',
                allowEmptyResults: true,
                keepLongStdio: true
            )
    
            archiveArtifacts(
                artifacts: 'wdio_qei/wdio/reports/junit/*.xml',
                allowEmptyArchive: true,
                fingerprint: true
            )
        }
    
        cleanup {
            // 🔥 CLEAN ONLY AFTER JENKINS IS DONE
            cleanWs(
                patterns: [[pattern: 'wdio_qei/wdio/reports/**', type: 'INCLUDE']]
            )
        }
    }

}
