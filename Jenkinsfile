pipeline {
    agent any

    environment {
        DOCKER_HUB = "jnjaramillom/curso-devops-lab3"
        VERSION = "1.0.${BUILD_NUMBER}"
    }

    stages {

        stage('Install') {
            agent { docker { image 'node:20' } }
            steps {
                sh 'npm ci'
            }
        }

        stage('Build') {
            agent { docker { image 'node:20' } }
            steps {
                sh 'npm run build'
            }
        }

        stage('Test') {
            agent { docker { image 'node:20' } }
            steps {
                sh 'npm test || true'
            }
        }

        stage('SonarQube') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    sonar-scanner \
                    -Dsonar.projectKey=curso-devops-lab3 \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://localhost:9000
                    """
                }
            }
        }
    }
}