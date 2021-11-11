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
                        withEnv(['DATREE_TOKEN=CUuBJnjHgwVYGi5EZQPLr4']) {
                            sh '''
                            helm datree test myapp/
                            '''
                        }
                    }
                }
            }
        }

        stage("Push helm charts to Nexus") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
                        dir('kubernetes') {
                            sh '''
                            helmversion=$(helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                            tar -czvf myapp-${helmversion}.tgz myapp/
                            curl -u admin:$docker_password http://10.0.153.248:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                        }
                    }
                }
            }
        }

        stage("Manual approval") {
            steps {
                script {
                    timeout(10) {
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: "Deploy")
                    }
                }
            }
        }

        stage("Deploying application of k8s cluster") {
            steps {
                script {
                    withCredentials([kubeconfigFile(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                        dir('kubernetes') {
                            sh 'helm upgrade --install --set image.repository="10.0.153.248:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/'
                        }
                    }
                }
            }
        }

        stage("Verify app deployment") {
            steps {
                script {
                    withCredentials([kubeconfigFile(credentialsId: 'kubernetes-config', variable: 'KUBECONFIG')]) {
                        sh 'kubectl run curl --image curlimages/curl -i --rm --restart Never -- curl myjavaapp-myapp:8080'
                    }
                }
            }
        }
    }
}
