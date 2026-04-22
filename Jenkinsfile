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

        stage('Install dependencies') {
            agent { docker { image 'node:20'; reuseNode true } }
            steps { sh 'npm install' }
        }

        stage('Build') {
            agent { docker { image 'node:20'; reuseNode true } }
            steps { sh 'npm run build' }
        }

        stage('Test') {
            agent { docker { image 'node:20'; reuseNode true } }
            steps { sh 'npm test || true' }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    sh "docker rm -f sonar_scanner_tmp || true"
                    withSonarQubeEnv('sonar-server') {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh '''
                            docker run --name sonar_scanner_tmp \
                                -e SONAR_HOST_URL=http://host.docker.internal:8084 \
                                -e SONAR_TOKEN=${SONAR_TOKEN} \
                                -v ${WORKSPACE}:/usr/src \
                                sonarsource/sonar-scanner-cli \
                                -Dsonar.projectKey=curso-devops-lab3 \
                                -Dsonar.sources=. \
                                -Dsonar.working.directory=/tmp/.scannerwork
                            '''
                            sh "mkdir -p .scannerwork"
                            sh "docker cp sonar_scanner_tmp:/tmp/.scannerwork/report-task.txt .scannerwork/report-task.txt"
                        }
                    }
                }
            }
            post { always { sh "docker rm -f sonar_scanner_tmp || true" } }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t $DOCKER_HUB:latest -t $DOCKER_HUB:$VERSION ."
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
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
                script {
                    // Aplicar cambios generales
                    sh "docker run --rm -v /home/jjaramillo/.kube:/root/.kube -v \$(pwd):/app -w /app bitnami/kubectl:latest --insecure-skip-tls-verify apply -f kubernetes.yaml"
                    
                    // Objetivo 1.h: Actualizar el deployment con el build number específico
                    sh "docker run --rm -v /home/jjaramillo/.kube:/root/.kube bitnami/kubectl:latest --insecure-skip-tls-verify set image deployment/app-deployment app-container=ghcr.io/jnjaramillom-sketch/curso-devops-lab3:${env.BUILD_NUMBER} -n jjaramillo"
                }
            }
        }
    }
}