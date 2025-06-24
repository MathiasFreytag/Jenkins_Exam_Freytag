pipeline {
    agent any

    environment {
        DOCKER_HUB = credentials('dockerhub-credentials') // erzeugt: DOCKER_HUB_USR + DOCKER_HUB_PSW
    }

    stages {
        stage('Login to DockerHub') {
            steps {
                sh 'echo "$DOCKER_HUB_PSW" | docker login -u "$DOCKER_HUB_USR" --password-stdin'
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    def services = ['cast-service', 'movie-service']
                    for (svc in services) {
                        def image = "docker.io/${DOCKER_HUB_USR}/${svc}:latest"
                        sh """
                            echo "Building ${svc}..."
                            docker build -t ${image} ./${svc}
                            docker push ${image}
                        """
                    }
                }
            }
        }

        stage('Deploy to dev') {
            steps {
                deployToEnv('dev')
            }
        }

        stage('Deploy to qa') {
            steps {
                deployToEnv('qa')
            }
        }

        stage('Deploy to staging') {
            steps {
                deployToEnv('staging')
            }
        }

        stage('Deploy to prod (if master)') {
            steps {
                script {
                    def branch = env.GIT_BRANCH ?: sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    echo "Detected Jenkins branch: ${branch}"

                    if (branch.contains('master')) {
                        input message: 'Deploy to production?', ok: 'Yes, deploy'
                        deployToEnv('prod')()
                    } else {
                        echo "Skipping production deployment – not on master branch."
                    }
                }
            }
        }
    }
}

def deployToEnv(envName) {
    return {
        script {
            def services = ['cast-service', 'movie-service']

            withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                env.KUBECONFIG = "${KUBECONFIG_FILE}"

                for (svc in services) {
                    def image = "docker.io/${env.DOCKER_HUB_USR}/${svc}:latest"
                    def chartPath = "./charts"

                    sh """
                        echo "Deploying ${svc} to ${envName}..."
                        helm upgrade --install ${svc} ${chartPath} \
                          --namespace ${envName} \
                          --create-namespace \
                          --set image.repository=${image} \
                          --set image.tag=latest
                    """
                }
            }
        }
    }
}
