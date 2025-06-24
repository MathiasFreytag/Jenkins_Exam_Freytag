pipeline {
    agent any

    environment {
        DOCKER_HUB = credentials('dockerhub-credentials')
        KUBECONFIG_FILE = credentials('config')
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
                deployToEnv('dev', 'ClusterIP')
            }
        }

        stage('Deploy to qa') {
            steps {
                deployToEnv('qa', 'ClusterIP')
            }
        }

        stage('Deploy to staging') {
            steps {
                deployToEnv('staging', 'ClusterIP')
            }
        }

        stage('Manual Approval') {
            when {
                expression {
                    echo "Detected GIT_BRANCH: ${env.GIT_BRANCH}"
                    return env.GIT_BRANCH == 'origin/master'
                }
            }
            steps {
                input message: 'Deploy to production?', ok: 'Yes, deploy'
            }
        }

        stage('Deploy to prod') {
            when {
                expression {
                    return env.GIT_BRANCH == 'origin/master'
                }
            }
            steps {
                deployToEnv('prod', 'NodePort')
            }
        }
    }
}

def deployToEnv(envName, serviceType) {
    return {
        withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
            script {
                def services = ['cast-service', 'movie-service']
                def nodePorts = ['cast-service': 30007, 'movie-service': 30008]

                sh '''
                    mkdir -p /var/lib/jenkins/.kube
                    cp "$KUBECONFIG_FILE" /var/lib/jenkins/.kube/config
                    export KUBECONFIG=/var/lib/jenkins/.kube/config
                '''

                for (svc in services) {
                    def imageRepo = "docker.io/${DOCKER_HUB_USR}/${svc}"
                    def chartPath = "./charts"
                    def nodePortFlag = serviceType == "NodePort" ? "--set service.nodePort=${nodePorts[svc]}" : ""

                    sh """
                        echo "Deploying ${svc} to ${envName}..."
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        helm upgrade --install ${svc} ${chartPath} \
                          --namespace ${envName} \
                          --create-namespace \
                          --set image.repository=${imageRepo} \
                          --set image.tag=latest \
                          --set service.type=${serviceType} \
                          ${nodePortFlag}
                    """
                }
            }
        }
    }
}
