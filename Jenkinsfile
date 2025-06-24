pipeline {
    agent any

    environment {
        DOCKER_HUB = credentials('dockerhub-credentials') // erzeugt: DOCKER_HUB_USR + DOCKER_HUB_PSW
        KUBECONFIG_FILE = credentials('kubeconfig-file')  // Kubernetes config als Datei
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

        stage('Manual Approval for Prod') {
            when {
                branch 'master'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Yes, deploy'
            }
        }

        stage('Deploy to prod') {
            when {
                branch 'master'
            }
            steps {
                deployToEnv('prod')
            }
        }
    }
}

def deployToEnv(envName) {
    return {
        script {
            def services = ['cast-service', 'movie-service']

            // Verwende File-Credential
            sh '''
                mkdir -p ~/.kube
                cp "$KUBECONFIG_FILE" ~/.kube/config
                export KUBECONFIG=~/.kube/config
            '''

            for (svc in services) {
                def image = "docker.io/${DOCKER_HUB_USR}/${svc}:latest"
                def chartPath = "./charts/${svc}"

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
