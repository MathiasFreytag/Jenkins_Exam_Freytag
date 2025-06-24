pipeline {
    agent any

    environment {
        DOCKER_HUB = credentials('dockerhub-credentials') // erzeugt: DOCKER_HUB_USR + DOCKER_HUB_PSW
        KUBECONFIG_FILE = credentials('config')  // Kubernetes config als Datei
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
                    def branch = env.GIT_BRANCH
                    echo "Detected Jenkins branch: ${branch}"

                    if (branch?.contains('master')) {
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
            def kubeConfigPath = "${env.WORKSPACE}/kubeconfig.yaml"

            // Kubeconfig vorbereiten
            sh """
                cp "$KUBECONFIG_FILE" ${kubeConfigPath}
                export KUBECONFIG=${kubeConfigPath}
            """

            for (svc in services) {
                def image = "docker.io/${DOCKER_HUB_USR}/${svc}:latest"
                def chartPath = "./charts"

                sh """
                    echo "Deploying ${svc} to ${envName}..."
                    export KUBECONFIG=${kubeConfigPath}
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
