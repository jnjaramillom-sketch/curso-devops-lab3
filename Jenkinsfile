pipeline {
    agent any

    environment {
        IMAGE_NAME = "jnjaramillom/curso-devops-lab3"
        GH_IMAGE = "ghcr.io/jnjaramillom/curso-devops-lab3"
        VERSION = "1.0.${BUILD_NUMBER}"
    }

    stages {

        stage('Install dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run tests') {
            steps {
                sh 'npm run test:cov'
            }
        }

        stage('Build app') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Build Docker image') {
            steps {
                sh """
                docker build -t $IMAGE_NAME:latest \
                             -t $IMAGE_NAME:$VERSION \
                             -t $IMAGE_NAME:${BUILD_NUMBER} .
                """
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh """
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $IMAGE_NAME:latest
                    docker push $IMAGE_NAME:$VERSION
                    docker push $IMAGE_NAME:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                    kubectl apply -f kubernetes.yaml
                    """
                }
            }
        }
    }
}
