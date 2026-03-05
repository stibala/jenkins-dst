pipeline {
    environment {
        // Declaration of environment variables
        DOCKER_ID    = "adamstibal"                        // replace this with your docker hub username/id
        DOCKER_IMAGE = "datascientestapi"
        DOCKER_TAG   = "v.${BUILD_ID}.0"                // tags increment with each build
    }

    agent any   // Jenkins can pick any available agent

    stages {
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                        docker rm -f jenkins || true
                        docker build -t ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG} .
                        sleep 6
                    '''
                }
            }
        }

        stage('Docker Run') {
            steps {
                script {
                    sh '''
                        docker run -d -p 80:80 --name jenkins ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
                        sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                        curl http://localhost
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")   // secret text credential in Jenkins
            }
            steps {
                script {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u ${DOCKER_ID} --password-stdin
                        docker push ${DOCKER_ID}/${DOCKER_IMAGE}:${DOCKER_TAG}
                    '''
                }
            }
        }

        stage('Deployment in dev') {
            environment {
                KUBECONFIG = credentials("config")   // secret file credential containing kubeconfig
            }
            steps {
                script {
                    sh '''
                        rm -rf .kube
                        mkdir -p .kube
                        echo "$KUBECONFIG" > .kube/config
                        cp fastapi/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                        helm upgrade --install app fastapi \
                            --values=values.yml \
                            --namespace dev \
                            --create-namespace
                    '''
                }
            }
        }

        stage('Deployment in staging') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                        rm -rf .kube
                        mkdir -p .kube
                        echo "$KUBECONFIG" > .kube/config
                        cp fastapi/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                        helm upgrade --install app fastapi \
                            --values=values.yml \
                            --namespace staging \
                            --create-namespace
                    '''
                }
            }
        }

        stage('Deployment in prod') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                // Manual approval required before production deployment
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you want to deploy in production?',
                          ok:      'Yes, deploy to production'
                }

                script {
                    sh '''
                        rm -rf .kube
                        mkdir -p .kube
                        echo "$KUBECONFIG" > .kube/config
                        cp fastapi/values.yaml values.yml
                        sed -i "s+tag:.*+tag: ${DOCKER_TAG}+g" values.yml
                        helm upgrade --install app fastapi \
                            --values=values.yml \
                            --namespace prod \
                            --create-namespace
                    '''
                }
            }
        }
    }

    // Optional: you could add a post section here
    // post {
    //     always {
    //         echo "Pipeline finished - Build #${BUILD_ID}"
    //     }
    //     success {
    //         echo "Deployment successful!"
    //     }
    //     failure {
    //         echo "Pipeline failed!"
    //     }
    // }
}