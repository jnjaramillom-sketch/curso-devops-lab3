pipeline {
    agent {
        node {
            label 'built-in'
            customWorkspace '/home/jjaramillo/projects/curso-devops-lab3'
        }
    }

    environment {
        DOCKER_HUB = "jnjaramillom/curso-devops-lab3"
        GHCR = "ghcr.io/jnjaramillom-sketch/curso-devops-lab3"
        VERSION = "1.0.${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                // Con customWorkspace definido arriba, checkout scm descargará todo aquí
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh '''
                docker run --rm \
                -v ${WORKSPACE}:/app \
                -w /app \
                node:20 npm install
                '''
            }
        }

        stage('Build') {
            steps {
                sh '''
                docker run --rm \
                -v ${WORKSPACE}:/app \
                -w /app \
                node:20 npm run build
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                docker run --rm \
                -v ${WORKSPACE}:/app \
                -w /app \
                node:20 npm test || true
                '''
            }
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
                            sh "docker cp sonar_scanner_tmp:/tmp/.scannerwork/report-task.txt .scannerwork/report-task.txt || true"
                        }
                    }
                }
            }
            post {
                always {
                    sh "docker rm -f sonar_scanner_tmp || true"
                }
            }
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
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_HUB:latest
                    docker push $DOCKER_HUB:$VERSION
                    '''
                }
            }
        }

        stage('Push GHCR') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                    sh '''
                    echo $TOKEN | docker login ghcr.io -u jnjaramillom-sketch --password-stdin
                    docker tag $DOCKER_HUB:latest $GHCR:latest
                    docker tag $DOCKER_HUB:$VERSION $GHCR:$VERSION
                    docker push $GHCR:latest
                    docker push $GHCR:$VERSION
                    '''
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                script {
                    sh '''
                    docker run --rm \
                    -v /var/jenkins_home/.kube:/root/.kube \
                    -v ${WORKSPACE}:/app \
                    -w /app \
                    bitnami/kubectl:latest \
                    --insecure-skip-tls-verify \
                    apply -f kubernetes.yaml
                    '''
                }
            }
        }
    }
}