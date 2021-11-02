pipeline {
    agent any
    environment {
        VERSION = "${env.BUILD_ID}"
    }

    stages {
        stage("Sonar validity check") {
            agent {
                docker {
                    image 'openjdk:11'
                }
            }
            steps {
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh 'chmod +x gradlew'
                        sh './gradlew sonarqube'
                    }

                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failures: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage("Docker build and push") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                        sh '''
                        docker build -t 10.0.153.248:8083/springapp:${VERSION} .
                        docker login -u admin -p $docker_password 10.0.153.248:8083
                        docker push 10.0.153.248:8083/springapp:${VERSION}
                        docker rmi 10.0.153.248:8083/springapp:${VERSION}
                        '''
                    }
                }
            }
        }

        stage("Identify misconfigs using datree in helm charts") {
            steps {
                script {
                    dir('kubernetes') {
                        sh 'helm datree test myapp/'
                    }
                }
            }
        }
    }
}
