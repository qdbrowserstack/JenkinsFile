pipeline {
    agent any
    
    options {
        skipDefaultCheckout(true)
        timeout(time: 30, unit: 'MINUTES')
    }
    
    parameters {
        string(name: 'TEST_CLASS', defaultValue: '', description: 'Specific test class to run (leave empty for TestClass1,TestClass2)')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment for testing')
        booleanParam(name: 'SKIP_FAILED_RERUN', defaultValue: false, description: 'Skip rerunning failed tests')
    }
    
    environment {
        MAVEN_PATH = '/opt/homebrew/bin/mvn'
        PROJECT_DIR = '/Users/sanjayshukla/Desktop/Practice/my-junit-demo'
        DEFAULT_TEST_CLASSES = 'TestClass1,TestClass2'
        FAILED_TESTS_FILE = 'failed-tests.txt'
    }

    stages {
        stage('Setup') {
            steps {
                script {
                    echo ":rocket: Child Job 1 - Setting up for TestClass1 & TestClass2"
                    
                    sh """
                        rm -rf ${WORKSPACE}/*
                        cp -r ${PROJECT_DIR}/* ${WORKSPACE}/
                    """
                    
                    def testClasses = params.TEST_CLASS ?: env.DEFAULT_TEST_CLASSES
                    env.TARGET_TEST_CLASSES = testClasses
                    echo "Target test classes: ${env.TARGET_TEST_CLASSES}"
                }
            }
        }
        
        stage('Run All Tests') {
            steps {
                script {
                    echo ":test_tube: Stage 1: Running all test cases"
                    
                    try {
                        sh """
                            ${MAVEN_PATH} clean compile
                            
                            # Run tests for TestClass1,TestClass2
                            ${MAVEN_PATH} test -Dtest=${env.TARGET_TEST_CLASSES} \
                                -Dsurefire.reportFormat=xml \
                                -Dmaven.test.failure.ignore=true \
                                -Denv=${params.ENVIRONMENT}
                        """
                    } catch (Exception e) {
                        echo ":warning: Test execution had issues: ${e.message}"
                        unstable("Tests failed but continuing")
                    }
                }
            }
        }
        
        stage('Analyze Results') {
            steps {
                script {
                    echo ":bar_chart: Analyzing test results"
                    
                    // Extract failed tests - FIXED SYNTAX
                    sh """
                        > ${FAILED_TESTS_FILE}
                        
                        if [ -d "target/surefire-reports" ]; then
                            for xml_file in target/surefire-reports/TEST-*.xml; do
                                if [ -f "\$xml_file" ]; then
                                    # Extract failed test methods
                                    if grep -q 'failures="[1-9]' "\$xml_file"; then
                                        class_name=\$(basename "\$xml_file" .xml | sed 's/TEST-//')
                                        echo "\$class_name" >> ${FAILED_TESTS_FILE}
                                    fi
                                fi
                            done
                        fi
                        
                        echo "Failed tests identified:"
                        cat ${FAILED_TESTS_FILE} || echo "No failed tests found"
                    """
                }
            }
        }
        
        stage('Rerun Failed Tests') {
            when {
                allOf {
                    expression { !params.SKIP_FAILED_RERUN }
                    expression { fileExists('failed-tests.txt') }
                }
            }
            steps {
                script {
                    echo ":arrows_counterclockwise: Stage 2: Rerunning failed tests"
                    
                    if (fileExists(FAILED_TESTS_FILE)) {
                        def failedTests = readFile(FAILED_TESTS_FILE).trim()
                        
                        if (failedTests) {
                            echo "Rerunning failed tests: ${failedTests}"
                            
                            try {
                                sh """
                                    # Rerun failed tests
                                    ${MAVEN_PATH} test -Dtest=${failedTests.replace('\n', ',')} \
                                        -Dsurefire.reportFormat=xml \
                                        -Dmaven.test.failure.ignore=true \
                                        -Denv=${params.ENVIRONMENT}
                                """
                            } catch (Exception e) {
                                echo ":warning: Failed test rerun had issues: ${e.message}"
                            }
                        } else {
                            echo ":white_check_mark: No failed tests to rerun"
                        }
                    }
                }
            }
        }
        
        stage('Publish Results') {
            steps {
                script {
                    if (fileExists('target/surefire-reports')) {
                        junit testResults: '**/target/surefire-reports/*.xml',
                              allowEmptyResults: true
                        
                        archiveArtifacts artifacts: '**/target/surefire-reports/*.xml',
                                       allowEmptyArchive: true
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Child Job 1 execution completed"
            }
        }
        success {
            echo ":white_check_mark: Child Job 1 completed successfully!"
        }
        failure {
            echo ":x: Child Job 1 failed!"
        }
        unstable {
            echo ":warning: Child Job 1 completed with test failures"
        }
    }
}
