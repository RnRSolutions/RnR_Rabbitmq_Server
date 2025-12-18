pipeline {
    agent any

    environment {
        // Update these values as needed
        DOCKER_IMAGE_NAME = "nuwanRNR/rnr_rabbitmq_server"
        // Ensure this matches the OTP_VERSION in packaging/docker-image/Dockerfile if possible, 
        // though strict parity isn't always required for the *builder* if the release is compatible.
        ERLANG_BUILD_IMAGE = "erlang:27" 
    }

    stages {
        stage('SCM Checkout') {
            steps {
                retry(3) {
                    checkout scm
                }
            }
        }

        stage('Build Artifact') {
            agent {
                docker {
                    image "${ERLANG_BUILD_IMAGE}"
                    // map the workspace so we can write artifacts
                    reuseNode true 
                }
            }
            steps {
                script {
                    echo 'Installing build dependencies...'
                    // RabbitMQ Makefiles require rsync, zip, git, make, etc.
                    sh 'apt-get update && apt-get install -y rsync zip make git xz-utils'
                    
                    echo 'Building generic unix package...'
                    // This command runs the build defined in the Makefile
                    // It should produce a .tar.xz file in PACKAGES/
                    sh 'make package-generic-unix'
                }
            }
        }

        stage('Prepare Docker Context') {
            steps {
                script {
                    // Locate the generated artifact. 
                    // The Makefile defaults PACKAGES_DIR to $(abspath PACKAGES)
                    def artifacts = findFiles(glob: 'PACKAGES/rabbitmq-server-generic-unix-*.tar.xz')
                    
                    if (artifacts.length == 0) {
                        error "No build artifact found in PACKAGES/"
                    }
                    
                    def artifactPath = artifacts[0].path
                    echo "Found artifact: ${artifactPath}"
                    
                    // The Dockerfile in packaging/docker-image expects 'package-generic-unix.tar.xz'
                    // We copy the build artifact to that location.
                    sh "cp ${artifactPath} packaging/docker-image/package-generic-unix.tar.xz"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dir('packaging/docker-image') {
                        echo 'Building Docker image...'
                        sh "docker build -t ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} ."
                    }
                }
            }
        }
        
        // Uncomment/Modify for pushing to registry
        /*
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-login', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "docker login -u ${DOCKER_USER} -p '${DOCKER_PASS}'"
                    sh "docker push ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                }
            }
        }
        */
        
        stage('Deploy') {
             steps {
                 script {
                     echo 'Deploying container...'
                     // Stop and remove existing container (ignore errors if not existing)
                     sh 'docker stop rnr_rabbitmq_server || true'
                     sh 'docker rm rnr_rabbitmq_server || true'
                     
                     // Run the new container
                     // Note: RabbitMQ default ports are 5672, 15672 (mgmt)
                     sh "docker run -d --restart unless-stopped --name rnr_rabbitmq_server -p 5672:5672 -p 15672:15672 ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                 }
             }
        }

        stage('Cleanup') {
            steps {
                sh 'docker container prune -f'
                sh 'docker image prune -f' 
            }
        }
    }

    post {
        always {
            cleanWs()
            echo 'Pipeline finished'
        }
    }
}
