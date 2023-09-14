pipeline {
    agent any

    stages {
        stage('Checkout SCM') {
            steps {
                script {
                    def gitUrl = 'https://github.com/shantanudatarkar/AppCode.git'
                    checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'Github_id', url: gitUrl]]])
                }
            }
        }

        stage('Check Disk Space') {
            steps {
                script {
                    def freeSpace = sh(script: 'df -k / | tail -n 1 | awk \'{print $4}\'', returnStatus: true, returnStdout: true).trim()
                    def requiredSpace = 1024 * 1024 // Set the required space in KB (adjust as needed)
                    echo "Free disk space: ${freeSpace} KB"

                    if (freeSpace.isNumber() && freeSpace.toInteger() < requiredSpace) {
                        error "Not enough free disk space to proceed."
                    } else {
                        echo "There is enough free space. Proceeding with the pipeline."
                    }
                }
            }
        }

        stage('Build and Push') {
            agent {
                docker {
                    image 'cimg/node:20.3.1'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    def version = "build-${BUILD_NUMBER}"
                    echo "Building Docker image: ${version}"
                    sh "docker image prune -a --filter \"until=${30*24*3600}\" -f"
                    sh "docker build -t piyushsachdeva/todo-app:${version} ."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "echo \${DOCKER_PASSWORD} | docker login -u \${DOCKER_USERNAME} --password-stdin"
                        sh "docker push piyushsachdeva/todo-app:${version}"
                    }
                }
            }
        }

        stage('Update Manifest') {
            agent {
                docker {
                    image 'cimg/base:2023.06'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                script {
                    def TAG = BUILD_NUMBER.toInteger() - 1
                    def gitUrl = 'https://github.com/piyushsachdeva/kube_manifest.git'
                    def gitUserEmail = 'piyush.sachdeva055@gmail.com'
                    def gitUserName = 'piyushsachdeva'

                    sh "git clone ${gitUrl}"
                    sh "git config --global user.email '${gitUserEmail}'"
                    sh "git config --global user.name '${gitUserName}'"
                    dir('kube_manifest') {
                        echo "Current directory: ${pwd()}"
                        echo "Using TAG: ${TAG}"
                        sh "sed -i \"s/build-.*/build-${TAG}/g\" manifest/deployment.yaml"
                        sh "cat manifest/deployment.yaml"
                        sh "git add ."
                        sh "git commit -m 'new build with imgTag build-${TAG}'"
                        sh "git config credential.helper 'cache --timeout=120'"
                        sh "git push -q ${gitUrl} main"
                    }
                }
            }
        }
    }
}
