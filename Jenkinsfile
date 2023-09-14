pipeline {
    agent any

    stages {
        stage('Check Disk Space') {
            steps {
                script {
                    def freeSpace = sh(script: "df -h / | awk 'NR==2{print \$4}'", returnStdout: true).trim()
                    echo "Free disk space: ${freeSpace}"

                    // Define a threshold (e.g., 1 GB) to determine if there's enough space
                    def threshold = "1G"

                    if (freeSpace >= threshold) {
                        echo "There is enough free space. Proceeding with the pipeline."
                    } else {
                        error "Insufficient disk space. Aborting the pipeline."
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
                checkout scm
                script {
                    def version = "build-${BUILD_NUMBER}"
                    echo "Building Docker image: ${version}"
                    // Clean up Docker images older than a certain threshold (e.g., 7 days)
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
                checkout scm
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
