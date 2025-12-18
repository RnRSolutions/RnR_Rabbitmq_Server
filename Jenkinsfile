pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = "nuwanRNR/rnr_rabbitmq_server"
        ERLANG_BUILD_IMAGE = "erlang:27"
    }

    stages {

        stage('SCM Checkout') {
            steps {
                retry(3) {
                    withCredentials([string(credentialsId: 'GitHubLogin', variable: 'GitHubLogin')]) {
                        // Updated to match the current project repo
                        git branch: 'CI/CD',
                            url: "https://${GitHubLogin}@github.com/RnRSolutions/RnR_Rabbitmq_Server.git"
                    }
                }
            }
        }

        stage('Prepare Secrets') {
            steps {
                // Ensure these credentials exist in Jenkins
                withCredentials([
                    string(credentialsId: 'RABBITMQ_DEFAULT_PASS', variable: 'RABBITMQ_DEFAULT_PASS'),
                    string(credentialsId: 'RABBITMQ_ERLANG_COOKIE', variable: 'RABBITMQ_ERLANG_COOKIE'),
                ]) {
                    sh """
                        echo "RABBITMQ_DEFAULT_USER=root" > .env
                        echo "RABBITMQ_DEFAULT_PASS=$RABBITMQ_DEFAULT_PASS" >> .env
                        echo "RABBITMQ_ERLANG_COOKIE=$RABBITMQ_ERLANG_COOKIE" >> .env
                    """
                }
            }
        }

        stage('Build Artifact') {
            steps {
                script {
                    echo 'Building RabbitMQ artifact inside Erlang container...'
                    // We use docker run manually because the Jenkins Docker Pipeline plugin is missing
                    // We mount the current workspace into the container to perform the build
                    sh """
                        docker run --rm -v \$(pwd):/app -w /app ${ERLANG_BUILD_IMAGE} bash -c "
                            apt-get update && apt-get install -y rsync zip make git xz-utils &&
                            make package-generic-unix
                        "
                    """
                }
            }
        }

        stage('Prepare Docker Context') {
            steps {
                script {
                    // Find the generated .tar.xz file
                    def artifacts = findFiles(glob: 'PACKAGES/rabbitmq-server-generic-unix-*.tar.xz')
                    if (artifacts.length == 0) {
                        error "No build artifact found in PACKAGES/"
                    }
                    def artifactPath = artifacts[0].path
                    echo "Found artifact: ${artifactPath}"
                    
                    // Copy to docker context
                    sh "cp ${artifactPath} packaging/docker-image/package-generic-unix.tar.xz"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'ls -la packaging/docker-image' // Verify artifact exists
                script {
                    dir('packaging/docker-image') {
                        sh "docker build -t ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} ."
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    echo 'Deploying container...'
                    sh 'docker stop rnr_rabbitmq_server || true'
                    sh 'docker rm rnr_rabbitmq_server || true'
                    
                    // Using --network host and .env file as requested
                    sh "docker run -d --restart unless-stopped --name rnr_rabbitmq_server --network host --env-file .env ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                    
                    sleep 5
                    sh "docker ps -a"
                    sh "docker logs rnr_rabbitmq_server"
                }
            }
        }

        stage('Docker Cleanup') {
            steps {
                sh '''
                docker container prune -f
                docker image prune -a -f
                docker builder prune -a -f
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished'
            sh 'rm -f .env' 
        }
    }
}
