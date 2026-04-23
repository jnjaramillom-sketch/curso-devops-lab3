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
                // Usamos ${env.BUILD_NUMBER} para que coincida con el Push
                sh "docker build -t jnjaramillom/curso-devops-lab3:latest -t jnjaramillom/curso-devops-lab3:${env.BUILD_NUMBER} ."
            }
        }

        stage('Push Docker Hub') {
            steps {
                script {
                    // Este 'dockerhub' debe ser igual al ID que pusiste en la configuración
                    docker.withRegistry('', 'dockerhub') {
                        sh "docker push jnjaramillom/curso-devops-lab3:${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                    docker run --rm \
                    --network devops-infra_default \
                    --add-host=host.docker.internal:host-gateway \
                    -v /var/jenkins_home/.kube:/root/.kube \
                    bitnami/kubectl:latest \
                    --kubeconfig=/root/.kube/config \
                    --insecure-skip-tls-verify=true \
                    set image deployment/curso-devops app=jnjaramillom/curso-devops-lab3:8
                    """
                }
            }
        }
        stage('Push GHCR') {
            steps {
                script {
                    // Usamos usernamePassword porque así es como creaste la credencial en Jenkins
                    withCredentials([usernamePassword(credentialsId: 'github-token-lab', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                        sh "echo \$GH_TOKEN | docker login ghcr.io -u \$GH_USER --password-stdin"
                        sh "docker tag jnjaramillom/curso-devops-lab3:latest ghcr.io/\$GH_USER/curso-devops-lab3:latest"
                        sh "docker tag jnjaramillom/curso-devops-lab3:latest ghcr.io/\$GH_USER/curso-devops-lab3:9"
                        sh "docker push ghcr.io/\$GH_USER/curso-devops-lab3:latest"
                        sh "docker push ghcr.io/\$GH_USER/curso-devops-lab3:9"
                    }
                }
            }
        }

        stage('Deploy Kubernetes') {
            steps {
                script {
                    sh """
                    docker run --rm \
                    --network devops-infra_default \
                    --add-host=host.docker.internal:host-gateway \
                    -v /var/jenkins_home/.kube:/root/.kube \
                    -v /home/jjaramillo/projects/curso-devops-lab3:/app \
                    -w /app \
                    bitnami/kubectl:latest \
                    --kubeconfig=/root/.kube/config \
                    --insecure-skip-tls-verify=true \
                    apply -f kubernetes.yaml
                    """
                }
            }
        }
    }
}