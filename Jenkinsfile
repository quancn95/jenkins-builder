def scmUrl = scm.getUserRemoteConfigs()[0].getUrl()
pipeline {
    //Definition of env variables that can be used throughout the pipeline job
    environment {
        // Unity tool installation
        UNITY_EXECUTABLE_LINUX = '/opt/unity/Editor/Unity'
        // Unity data
        UNITY_ID_EMAIL = credentials('unity-email')
        UNITY_ID_PASSWORD = credentials('unity-password')
        UNITY_ID_LICENSE = credentials('unity-license-windows')
        UNITY_ID_LICENSE_LINUX = credentials('unity-license-docker-linux')

        CI = true
        ARTIFACTORY_ACCESS_TOKEN = credentials('jfrog-access-token')
        ARTIFACTORY_URL = credentials('jfrog-url')
        ARTIFACTORY_PATH = "unity-internal/"
    }
    //Options: add timestamp to job logs and limiting the number of builds to be kept.
    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: "10"))
        skipDefaultCheckout(true)
    }

    agent { label 'ubuntu' }
    stages {
        stage('Run Build and Test') {
            parallel {
                stage('Windows Target') {
                    environment {
                        TEST_PLATFORM = 'playmode'
                        TESTING_TYPE = 'JUNIT'
                        BUILD_TARGET = 'StandaloneWindows64'
                        BUILD_PATH = "${env.WORKSPACE}/Builds/${env.BUILD_TARGET}"
//                         PROJECT_PATH = "${PROJECT_NAME}-wins"
//                         BUILD_NAME = "${PROJECT_NAME}-${env.BUILD_NUMBER}"

                    }
                    stages {
                        stage('Clone repository') {
                            steps{
                                dir("${env.BUILD_TARGET}") {
                                    git url: "${scmUrl}", branch: "$env.BRANCH_NAME", credentialsId: "github-authentication"
                                    sh 'ls -al'
                                }
                                script{
                                    env.PROJECT_NAME = sh(script: 'grep -oPm1 "(?<=productName: )[^ ]+" ${BUILD_TARGET}/ProjectSettings/ProjectSettings.asset',
                                                                                    returnStdout: true).trim()
                                    env.PROJECT_VERSION = sh(script: 'grep -oPm1 "(?<=bundleVersion: )[^ ]+" ${BUILD_TARGET}/ProjectSettings/ProjectSettings.asset',
                                                                                            returnStdout: true).trim()
                                    echo "${PROJECT_NAME}"
                                    echo "${PROJECT_VERSION}"
                                }
                            }
                        }
                        stage('Build') {
                            agent {
                                docker {
                                    label 'ubuntu && docker'
                                    image 'unityci/editor:ubuntu-2021.3.2f1-windows-mono-1'
                                    reuseNode true
                                    args '-u root:root --privileged'
                                }
                            }
                            stages{
                                stage('Activate License') {
                                    steps {
                                        script {
                                            try {
                                                echo "Activate License..."
                                                sh '${UNITY_EXECUTABLE_LINUX} -quit -nographics -batchmode \
                                                                              -username ${UNITY_ID_EMAIL} \
                                                                              -password ${UNITY_ID_PASSWORD} \
                                                                              -manualLicenseFile ${UNITY_ID_LICENSE_LINUX} -logFile'
                                            } catch (e) {
                                                echo "JOB FAILED: Load license."
                                            }
                                        }
                                    }
                                }
                                stage('Test Application') {
                                    steps {
                                        script {
                                            echo "Test App..."
                                            sh '''
                                               ${UNITY_EXECUTABLE_LINUX} -batchmode -nographics -runTests \
                                                                         -testPlatform ${TEST_PLATFORM} \
                                                                         -testResults ${WORKSPACE}/${BUILD_TARGET}/${TEST_PLATFORM}-results.xml \
                                                                         -projectPath ${WORKSPACE}/${BUILD_TARGET} \
                                                                         -enableCodeCoverage \
                                                                         -coverageResultsPath \
                                                                         ${WORKSPACE}/${BUILD_TARGET}/${TEST_PLATFORM}-coverage \
                                                                         -coverageOptions "generateAdditionalMetrics;generateHtmlReport;generateHtmlReportHistory;generateBadgeReport;" \
                                                                         -debugCodeOptimization -logFile /dev/stdout
                                              '''
                                        }
                                    }
                                }
                                stage('Build Application') {
                                    steps {
                                        script {
                                            echo "create Application output folder..."
                                            echo "Buld App..."
                                            sh 'echo ${PROJECT_NAME}'
                                            sh 'echo ${PROJECT_VERSION}'
                                            sh '''
                                               ${UNITY_EXECUTABLE_LINUX} -quit -batchmode -nographics \
                                                                         -projectPath ${WORKSPACE}/${BUILD_TARGET} \
                                                                         -buildTarget ${BUILD_TARGET} \
                                                                         -customBuildTarget ${BUILD_TARGET} \
                                                                         -customBuildName ${PROJECT_NAME} \
                                                                         -customBuildPath ${WORKSPACE}/${BUILD_TARGET}/Builds/${BUILD_NUMBER}/ \
                                                                         -executeMethod BuildScript.PerformBuild \
                                                                         -logFile /dev/stdout
                                               '''
                                            sh 'tar -czf ${BUILD_TARGET}/${PROJECT_NAME}-${PROJECT_VERSION}.tar.gz -C ${WORKSPACE}/${BUILD_TARGET}/Builds/${BUILD_NUMBER}/ .'
                                            sh 'ls -al ${WORKSPACE}/${BUILD_TARGET}/Builds'
                                        }
                                    }
                                }
                            }
                        }

                        stage('Upload to Artifactory') {
                            agent {
                                docker {
                                    label 'docker'
                                    image 'releases-docker.jfrog.io/jfrog/jfrog-cli-v2:2.2.0'
                                    reuseNode true
                                    args '-u root:root --privileged'
                                }
                            }
                            steps {
                                sh 'ls -al'
                                sh 'pwd'
                                sh '''
                                    jfrog rt upload --url ${ARTIFACTORY_URL}/artifactory/ \
                                                    --access-token ${ARTIFACTORY_ACCESS_TOKEN} \
                                                    ${WORKSPACE}/${BUILD_TARGET}/${PROJECT_NAME}-${PROJECT_VERSION}.tar.gz \
                                                    ${ARTIFACTORY_PATH}
                                   '''
                            }
                        }
                    }
                }
                stage('MacOS Target') {
                    environment {
                        TEST_PLATFORM = 'playmode'
                        TESTING_TYPE = 'JUNIT'
                        BUILD_TARGET = 'StandaloneOSX'
                        BUILD_PATH = "${env.WORKSPACE}/Builds/${BUILD_TARGET}/"
//                         PROJECT_PATH = "${PROJECT_NAME}-mac"
//                         BUILD_NAME = "${PROJECT_NAME}-${env.BUILD_NUMBER}"
                    }
                    stages {
                        stage('Clone repository') {
                            steps{
                                dir("${env.BUILD_TARGET}") {
                                    git url: "${scmUrl}", branch: "$env.BRANCH_NAME", credentialsId: "github-authentication"
                                    sh 'ls -al'
                                }
                                script{
                                    env.PROJECT_NAME = sh(script: 'grep -oPm1 "(?<=productName: )[^ ]+" ${BUILD_TARGET}/ProjectSettings/ProjectSettings.asset',
                                                                                    returnStdout: true).trim()
                                    env.PROJECT_VERSION = sh(script: 'grep -oPm1 "(?<=bundleVersion: )[^ ]+" ${BUILD_TARGET}/ProjectSettings/ProjectSettings.asset',
                                                                                            returnStdout: true).trim()
                                    echo "${PROJECT_NAME}"
                                    echo "${PROJECT_VERSION}"
                                }
                            }
                        }
                        stage('Build') {
                            agent {
                                docker {
                                    label 'ubuntu && docker'
                                    image 'unityci/editor:ubuntu-2021.3.2f1-mac-mono-1'
                                    reuseNode true
                                    args '-u root:root --privileged'
                                }
                            }
                            stages{
                                stage('Activate License') {
                                    steps {
                                        script {
                                            try {
                                                echo "Activate License..."
                                                sh '${UNITY_EXECUTABLE_LINUX} -quit -nographics -batchmode \
                                                                              -username ${UNITY_ID_EMAIL} \
                                                                              -password ${UNITY_ID_PASSWORD} \
                                                                              -manualLicenseFile ${UNITY_ID_LICENSE_LINUX} -logFile'
                                            } catch (e) {
                                                echo "JOB FAILED: Load license."
                                            }
                                        }
                                    }
                                }
                                stage('Test Application') {
                                    steps {
                                        script {
                                            echo "Test App..."
                                            sh '''
                                               ${UNITY_EXECUTABLE_LINUX} -batchmode -nographics -runTests \
                                                                         -testPlatform ${TEST_PLATFORM} \
                                                                         -testResults ${WORKSPACE}/${BUILD_TARGET}/${TEST_PLATFORM}-results.xml \
                                                                         -projectPath ${WORKSPACE}/${BUILD_TARGET} \
                                                                         -enableCodeCoverage \
                                                                         -coverageResultsPath \
                                                                         ${WORKSPACE}/${BUILD_TARGET}/${TEST_PLATFORM}-coverage \
                                                                         -coverageOptions "generateAdditionalMetrics;generateHtmlReport;generateHtmlReportHistory;generateBadgeReport;" \
                                                                         -debugCodeOptimization -logFile /dev/stdout
                                              '''
                                        }
                                    }
                                }
                                stage('Build Application') {
                                    steps {
                                        script {
                                            echo "create Application output folder..."
                                            echo "Buld App..."
                                            echo "${PROJECT_NAME}"
                                            echo "${PROJECT_VERSION}"
                                            sh '''
                                               ${UNITY_EXECUTABLE_LINUX} -quit -batchmode -nographics \
                                                                         -projectPath ${WORKSPACE}/${BUILD_TARGET} \
                                                                         -buildTarget ${BUILD_TARGET} \
                                                                         -customBuildTarget ${BUILD_TARGET} \
                                                                         -customBuildName ${PROJECT_NAME} \
                                                                         -customBuildPath ${WORKSPACE}/${BUILD_TARGET}/Builds/${BUILD_NUMBER}/ \
                                                                         -executeMethod BuildScript.PerformBuild \
                                                                         -logFile /dev/stdout
                                               '''
                                            sh 'tar -czf ${BUILD_TARGET}/${PROJECT_NAME}-${PROJECT_VERSION}.tar.gz -C ${WORKSPACE}/${BUILD_TARGET}/Builds/${BUILD_NUMBER}/ .'
                                            sh 'ls -al ${WORKSPACE}/${BUILD_TARGET}/Builds/'
                                        }
                                    }
                                }
                            }
                        }

                        stage('Upload to Artifactory') {
                            agent {
                                docker {
                                    label 'docker'
                                    image 'releases-docker.jfrog.io/jfrog/jfrog-cli-v2:2.2.0'
                                    reuseNode true
                                    args '-u root:root --privileged'
                                }
                            }
                            steps {
                                sh 'ls -al'
                                sh 'pwd'
                                sh '''
                                    jfrog rt upload --url ${ARTIFACTORY_URL}/artifactory/ \
                                                    --access-token ${ARTIFACTORY_ACCESS_TOKEN} \
                                                    ${WORKSPACE}/${BUILD_TARGET}/${PROJECT_NAME}-${PROJECT_VERSION}.tar.gz \
                                                    ${ARTIFACTORY_PATH}
                                   '''
                            }
                        }
                    }

                }
            }
        }

    }
    post {
        always {
            echo 'One way or another, I have finished'
            sh 'sudo chmod ugo+rwx -R ${WORKSPACE}'
            cleanWs()
//             deleteDir() /* clean up our workspace */
//             cleanWs(cleanWhenNotBuilt: false,
//                                         deleteDirs: true,
//                                         disableDeferredWipeout: true,
//                                         notFailBuild: true,
//                                         patterns: [[pattern: '.gitignore', type: 'INCLUDE'],
//                                                    [pattern: '.propsfile', type: 'EXCLUDE']])
        }
        success {
            echo 'I succeeded!'
        }
        unstable {
            echo 'I am unstable'
        }
        failure {
//             sh 'sudo chown vagrant:vagrant -R .'
            echo 'I failed'
        }
        changed {
            echo 'Things were different before...'
        }
    }
}
