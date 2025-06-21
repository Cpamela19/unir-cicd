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
                sh '''
                    docker rm -f unit-tests || true
                    mkdir -p results/unit
                    docker run --name unit-tests \
                        --env PYTHONPATH=/opt/calc \
                        -w /opt/calc \
                        calculator-app:latest \
                        /bin/sh -c "mkdir -p results/unit && pytest --cov --cov-report=xml:results/coverage.xml --cov-report=html:results/coverage --junit-xml=results/unit/unit_result.xml -m unit || true"
                    docker cp unit-tests:/opt/calc/results/unit/. results/unit/
                    docker rm unit-tests || true
                '''
                archiveArtifacts artifacts: 'results/unit/*.xml'
            }
        }
        stage('API Tests') {
            steps {
                sh '''
                    docker stop apiserver || true
                    docker rm -f apiserver || true
                    docker stop api-tests || true
                    docker rm -f api-tests || true
                    
                    docker network create calc-test-api || true
                    docker run -d --network calc-test-api \
                        --env PYTHONPATH=/opt/calc \
                        --env FLASK_APP=app/api.py \
                        --name apiserver \
                        -p 5000:5000 \
                        -w /opt/calc \
                        calculator-app:latest flask run --host=0.0.0.0
        
                    docker run --network calc-test-api --name api-tests \
                        --env PYTHONPATH=/opt/calc \
                        --env BASE_URL=http://apiserver:5000/ \
                        -w /opt/calc \
                        calculator-app:latest \
                        /bin/sh -c "mkdir -p results/api && pytest --junit-xml=results/api/api_result.xml -m api || true"
        
                    mkdir -p results/api
                    docker cp api-tests:/opt/calc/results/api/. results/api/
        
                    docker stop apiserver || true
                    docker rm --force apiserver || true
                    docker stop api-tests || true
                    docker rm --force api-tests || true
                    docker network rm calc-test-api || true
                '''
                archiveArtifacts artifacts: 'results/api/*.xml'
            }
        }
        stage('E2E Tests') {
            steps {
                sh '''
                    docker stop e2e-tests || true
                    docker rm --force e2e-tests || true
                    docker rm --force calc-web || true
                    
                    docker network create calc-test-e2e || true
                    docker run -d --network calc-test-e2e \
                        --env PYTHONPATH=/opt/calc \
                        --env FLASK_APP=app/api.py \
                        --name apiserver \
                        -p 5000:5000 \
                        -w /opt/calc \
                        calculator-app:latest flask run --host=0.0.0.0
        
                    docker run -d --network calc-test-e2e --name calc-web -p 80:80 calc-web
        
                    docker create --network calc-test-e2e --name e2e-tests cypress/included:4.9.0 --browser chrome || true
        
                    docker cp ./test/e2e/cypress.json e2e-tests:/cypress.json
                    docker cp ./test/e2e/cypress e2e-tests:/cypress
        
                    # Ejecuta los tests y asegura que los resultados estén en /results/e2e
                    docker start -a e2e-tests || true
        
                    # Asegura que la carpeta local exista y copia correctamente los resultados
                    mkdir -p results/e2e
                    docker cp e2e-tests:/results/e2e/. results/e2e/
        
                    docker rm --force apiserver || true
                    docker rm --force calc-web || true
                    docker rm --force e2e-tests || true
                    docker network rm calc-test-e2e || true
                '''
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
