pipeline {
    agent any

    environment {
        DOCKER_HUB = credentials('dockerhub-credentials') // erzeugt DOCKER_HUB_USR + DOCKER_HUB_PSW
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

        stage('Deploy to prod') {
            when {
                expression {
                    return getGitBranch() == "master"
                }
            }
            steps {
                input message: 'Deploy to production?', ok: 'Yes, deploy'
                script {
                    withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                        def services = ['cast-service', 'movie-service']
                        def ports = ['cast-service': 30007, 'movie-service': 30008]

                        sh '''
                            mkdir -p ~/.kube
                            cp "$KUBECONFIG_FILE" ~/.kube/config
                            export KUBECONFIG=~/.kube/config
                        '''

                        for (svc in services) {
                            def image = "docker.io/${DOCKER_HUB_USR}/${svc}:latest"
                            def chartPath = "./charts"
                            def nodePort = ports[svc]

                            sh """
                                echo "Deploying ${svc} to prod..."
                                helm upgrade --install ${svc} ${chartPath} \
                                  --namespace prod \
                                  --create-namespace \
                                  --set image.repository=${image} \
                                  --set image.tag=latest \
                                  --set service.type=NodePort \
                                  --set service.nodePort=${nodePort}
                            """
                        }
                    }
                }
            }
        }
    }
}

def deployToEnv(envName) {
    return {
        script {
            withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
                def services = ['cast-service', 'movie-service']

                sh '''
                    mkdir -p ~/.kube
                    cp "$KUBECONFIG_FILE" ~/.kube/config
                    export KUBECONFIG=~/.kube/config
                '''

                for (svc in services) {
                    def image = "docker.io/${DOCKER_HUB_USR}/${svc}:latest"
                    def chartPath = "./charts"

                    sh """
                        echo "Deploying ${svc} to ${envName}..."
                        helm upgrade --install ${svc} ${chartPath} \
                          --namespace ${envName} \
                          --create-namespace \
                          --set image.repository=${image} \
                          --set image.tag=latest \
                          --set service.type=ClusterIP
                    """
                }
            }
        }
    }
}

def getGitBranch() {
    def branch = sh(script: 'git rev-parse --abbrev-ref HEAD || echo HEAD', returnStdout: true).trim()
    echo "Detected Git branch: ${branch}"
    return branch ==~ /origin\/(master|main)/ ? "master" : branch
}
