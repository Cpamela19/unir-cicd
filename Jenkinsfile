pipeline {
    agent {
        docker {
            image 'docker:20.10.7-dind'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
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
                sh 'docker rm -f unit-tests || true'
                sh '''
                    docker run --name unit-tests \
                        --env PYTHONPATH=/opt/calc \
                        -w /opt/calc \
                        calculator-app:latest \
                        /bin/sh -c "mkdir -p results/unit && pytest --cov --cov-report=xml:results/coverage.xml --cov-report=html:results/coverage --junit-xml=results/unit/unit_result.xml -m unit || true"
                '''
                sh 'docker cp unit-tests:/opt/calc/results/unit ./'
                sh 'docker rm unit-tests || true'
                archiveArtifacts artifacts: 'results/unit/*.xml'
            }
        }
        stage('API Tests') {
            steps {
                sh 'docker network create calc-test-api || true'
                sh 'docker run -d --network calc-test-api --env PYTHONPATH=/opt/calc --name apiserver --env FLASK_APP=app/api.py -p 5000:5000 -w /opt/calc calculator-app:latest flask run --host=0.0.0.0'
                sh 'docker run --network calc-test-api --name api-tests --env PYTHONPATH=/opt/calc --env BASE_URL=http://apiserver:5000/ -w /opt/calc calculator-app:latest pytest --junit-xml=results/api_result.xml -m api  || true'
                sh 'docker cp api-tests:/opt/calc/results/api ./'
                sh 'docker stop apiserver || true'
                sh 'docker rm --force apiserver || true'
                sh 'docker stop api-tests || true'
                sh 'docker rm --force api-tests || true'
                sh 'docker network rm calc-test-api || true'
                archiveArtifacts artifacts: 'results/api/*.xml'
            }
        }
        stage('E2E Tests') {
            steps {
                sh 'docker network create calc-test-e2e || true'
                sh 'docker stop apiserver || true'
                sh 'docker rm --force apiserver || true'
                sh 'docker stop calc-web || true'
                sh 'docker rm --force calc-web || true'
                sh 'docker stop e2e-tests || true'
                sh 'docker rm --force e2e-tests || true'
                sh 'docker run -d --network calc-test-e2e --env PYTHONPATH=/opt/calc --name apiserver --env FLASK_APP=app/api.py -p 5000:5000 -w /opt/calc calculator-app:latest flask run --host=0.0.0.0'
                sh 'docker run -d --network calc-test-e2e --name calc-web -p 80:80 calc-web'
                sh 'docker create --network calc-test-e2e --name e2e-tests cypress/included:4.9.0 --browser chrome || true'
                sh 'docker cp ./test/e2e/cypress.json e2e-tests:/cypress.json'
                sh 'docker cp ./test/e2e/cypress e2e-tests:/cypress'
                sh 'docker start -a e2e-tests || true'
                sh 'docker cp e2e-tests:/results/e2e ./  || true'
                sh 'docker rm --force apiserver  || true'
                sh 'docker rm --force calc-web || true'
                sh 'docker rm --force e2e-tests || true'
                sh 'docker network rm calc-test-e2e || true'
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
