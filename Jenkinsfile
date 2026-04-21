pipeline {
    agent any

    environment {
        DOCKER_HUB = "jnjaramillom/curso-devops-lab3"
        GHCR = "ghcr.io/jnjaramillom-sketch/curso-devops-lab3"
        VERSION = "1.0.${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // Usamos el agente Docker nativo para Node.js
        // Esto monta automáticamente el workspace correctamente
        stage('Install dependencies') {
            agent {
                docker {
                    image 'node:20'
                    reuseNode true
                }
            }
            steps {
                sh 'npm install'
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:20'
                    reuseNode true
                }
            }
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:20'
                    reuseNode true
                }
            }
            steps {
                sh 'npm test || true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    // Aseguramos que la carpeta exista con permisos correctos antes de empezar
                    sh "mkdir -p .scannerwork && chmod 777 .scannerwork"
                    
                    withSonarQubeEnv('sonar-server') {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                            docker run --rm \
                                -e SONAR_HOST_URL=http://host.docker.internal:9000 \
                                -e SONAR_TOKEN=$SONAR_TOKEN \
                                -v ${WORKSPACE}:/usr/src \
                                sonarsource/sonar-scanner-cli \
                                -Dsonar.projectKey=curso-devops-lab3 \
                                -Dsonar.sources=. \
                                -Dsonar.working.directory=/usr/src/.scannerwork
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                docker build -t $DOCKER_HUB:latest \
                             -t $DOCKER_HUB:$VERSION .
                """
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_HUB:latest
                    docker push $DOCKER_HUB:$VERSION
                    """
                }
            }
        }

        stage('Push GHCR') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                    sh """
                    echo $TOKEN | docker login ghcr.io -u jnjaramillom-sketch --password-stdin
                    docker tag $DOCKER_HUB:latest $GHCR:latest
                    docker tag $DOCKER_HUB:$VERSION $GHCR:$VERSION
                    docker push $GHCR:latest
                    docker push $GHCR:$VERSION
                    """
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                // Asegúrate de que el agente tenga configurado el kubeconfig
                sh """
                kubectl set image deployment/app-deployment \
                app=$GHCR:$VERSION -n jjaramillo
                """
            }
        }
    }
}