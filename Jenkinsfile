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
        // Ajustamos el nombre al que configuraste en Jenkins
        SONAR_SERVER_NAME = 'SonarQube' 
    }

    stages {
        stage('Checkout') {
            steps {
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
                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                            sh '''
                            docker run --name sonar_scanner_tmp \
                                --network devops-infra_default \
                                -e SONAR_HOST_URL=http://sonarqube:9000 \
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
            // ... resto del código igual
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
                // Asegúrate de crear la credencial 'dockerhub' en Jenkins
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
                // Asegúrate de crear la credencial 'github-token-lab' en Jenkins (Secret Text)
                withCredentials([string(credentialsId: 'github-token-lab', variable: 'TOKEN')]) {
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