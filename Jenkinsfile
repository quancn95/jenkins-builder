def scmUrl = scm.getUserRemoteConfigs()[0].getUrl()

def parallelStagesMap

def generateStage(job) {
    return {
        stage("stage: ${job.name}") {
            environment {
                TEST_PLATFORM = 'playmode'
                TESTING_TYPE = 'JUNIT'
                BUILD_TARGET = '${job.target}'
                BUILD_PATH = "${env.WORKSPACE}/Builds/${env.BUILD_TARGET}"
//                         PROJECT_PATH = "${PROJECT_NAME}-wins"
//                         BUILD_NAME = "${PROJECT_NAME}-${env.BUILD_NUMBER}"

            }
            echo "${job.target}"

        }
    }
}
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
        stage('Create List of Stages to run in Parallel') {
            steps {
                script {
                    def list = [[name: "Windows", target: "StandaloneWindows64", image: "unityci/editor:ubuntu-2021.3.2f1-windows-mono-1"],
                                [name: "MacOs", target: "StandaloneOSX", image: "unityci/editor:ubuntu-2021.3.2f1-mac-mono-1"]]
                    parallelStagesMap = list.collectEntries {
                        ["${it.name}" : generateStage(it)]
                    }
                }
            }
        }
        stage('Clone repository') {
            steps {
                script {
                    parallel parallelStagesMap
                }
            }
        }
    }
}
