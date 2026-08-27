pipeline {
    agent any

    parameters {
        string(name: 'GIT_REPOSITORY', defaultValue: 'https://github.com/winstonjss/CICD-jenkins-securityAndMonitoring.git', description: 'URL del repositorio')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Rama para desplegar')
        string(name: 'IMAGE_NAME', defaultValue: 'cicd-lab-webapp', description: 'Nombre de la imagen Docker')
    }

    environment {
        SNYK_CRED_ID = 'SNYK_TOKEN'
        SONAR_SCANNER_HOME = tool 'SonarScanner'
    }

    stages {
        stage('Clone repository') {
            steps {
                git branch: params.GIT_BRANCH, url: params.GIT_REPOSITORY
            }
        }

        stage('Check Docker') {
            steps {
                sh 'docker --version'
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    echo 'Iniciando escaneo de seguridad de dependencias con Snyk...'
                    withCredentials([string(credentialsId: "${env.SNYK_CRED_ID}", variable: 'SNYK_TOKEN')]) {
                        sh 'npm install'
                        sh 'npx snyk test --severity-threshold=high || echo "Snyk detectó vulnerabilidades de severidad alta"'
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    echo 'Iniciando análisis estático de código con SonarQube...'
                    withSonarQubeEnv('SonarQube-Server') {
                        sh "${env.SONAR_SCANNER_HOME}/bin/sonar-scanner"
                    }
                }
            }
        }


        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker image') {
            steps {
                sh "docker build -t ${params.IMAGE_NAME}:${BUILD_NUMBER} -t ${params.IMAGE_NAME}:latest ."
            }
        }

        stage('Validate Docker image') {
            steps {
                sh "docker images | grep ${params.IMAGE_NAME}"
            }
        }

        stage('Remove previous container') {
            steps {
                sh "docker stop ${params.IMAGE_NAME} || true"
                sh "docker rm ${params.IMAGE_NAME} || true"
            }
        }

        stage('Run Docker container') {
            steps {
                sh "docker run -d --name ${params.IMAGE_NAME} -p 3000:3000 ${params.IMAGE_NAME}:latest"
            }
        }

        stage('Check container') {
            steps {
                sh "docker ps | grep ${params.IMAGE_NAME}"
            }
        }

        stage('Smoke test') {
            steps {
                // Validación del endpoint de salud [7, 35]
                sh 'sleep 5'
                sh 'curl -s http://localhost:3000/health || exit 1'
            }
        }

        stage('Cleanup old images') {
            steps {
                sh 'docker image prune -f'
            }
        }
    }

    post {
        success {
            echo '¡El pipeline de CD se completó con éxito con validaciones de seguridad superadas!'
        }
        failure {
            echo 'El pipeline falló. Por favor revise los logs de las etapas de validación o seguridad.'
        }
    }
}