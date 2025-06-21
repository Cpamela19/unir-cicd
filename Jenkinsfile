pipeline {
    agent {
        dockerfile true
    }
    
    stages {
        stage('Source') {
            steps {
                git 'https://github.com/Cpamela19/unir-cicd.git'
            }
        }
        stage('Build') {
            agent any
            steps {
                echo 'Building stage!'
                sh 'docker build -t calculator-app .'
                sh 'docker build -t calc-web ./web'
            }
        }
        stage('Unit tests') {
            steps {
                sh 'make test-unit'
                archiveArtifacts artifacts: 'results/unit/*.xml'
            }
        }
        stage('API Tests') {
            steps {
                sh 'make test-api'
                archiveArtifacts artifacts: 'results/api/*.xml'
            }
        }
        stage('E2E Tests') {
            steps {
                sh 'make test-e2e'
                archiveArtifacts artifacts: 'results/e2e/*.xml'
            }
        }
    }
    post {
        always {
            junit 'results/unit/*.xml'
            junit 'results/api/*.xml'
            junit 'results/e2e/*.xml'
            cleanWs()
        }
        failure {
            echo "Enviar correo: El pipeline '${env.JOB_NAME}' ha fallado en la ejecución #${env.BUILD_NUMBER}."
        }
    }
}
