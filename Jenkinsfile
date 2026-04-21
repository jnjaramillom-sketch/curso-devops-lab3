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
            steps {
                sh """
                docker run --rm \
                -v $WORKSPACE:/app \
                -w /app \
                node:20 npm install
                """
            }
        }

        stage('Build') {
            steps {
                sh """
                docker run --rm \
                    -v $PWD:/app \
                    -w /app \
                    node:20 npm run build
                """
            }
        }

        stage('Test') {
            steps {
                sh """
                docker run --rm \
                    -v $PWD:/app \
                    -w /app \
                    node:20 npm test || true
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                        sonar-scanner \
                        -Dsonar.projectKey=curso-devops-lab3 \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.login=$SONAR_TOKEN
                        """
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
                sh """
                kubectl set image deployment/app-deployment \
                app=$GHCR:$VERSION -n jjaramillo
                """
            }
        }
    }
}