// Prévoir un kubectl create configmap nginx-config --from-file=nginx_conf.conf

pipeline {
    environment {
        DOCKER_ID = "apooz"
        DOCKER_CAST_IMAGE = "jenkins-exam-cast_service"
        DOCKER_MOVIE_IMAGE = "jenkins-exam-movie_service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    agent any
    stages {
        stage('Init Docker Compose') {
            steps {
                script {
                    sh 'docker compose up -d'
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    sh '''
                        curl http://localhost:8080/api/v1/casts/docs
                        curl http://localhost:8080/api/v1/movies/docs
                    '''
                }
            }
        }

        stage('Stop Docker Compose') {
            steps {
                script {
                    sh 'docker compose down'
                }
            }
        }

        stage('Docker Build'){
            parallel {
                stage('Build Cast Service') {
                    steps {
                        script {
                            sh '''
                                docker rm -f cast
                                docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG cast-service
                            '''
                        }
                    }
                }
                stage('Build Movie Service') {
                    steps {
                        script {
                            sh '''
                                docker rm -f movie
                                docker build -t $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG movie-service
                            '''   
                        }
                    }
                }
            }
        }
        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            parallel {
                stage('Push Cast Service') {
                    steps {
                        script {
                            sh '''
                                docker login -u $DOCKER_ID -p $DOCKER_PASS
                                docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                            '''
                        }
                    }
                }
                stage('Push Movie Service') {
                    steps {
                        script {
                            sh '''
                                docker login -u $DOCKER_ID -p $DOCKER_PASS
                                docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                            '''
                        }
                    }
                }
            }
        }
        stage('Setup Kubernetes Config') {
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        ls -la
                        cat $KUBE_CONFIG > .kube/config
                        cat .kube/config
                    '''
                }
            }
        }
        stage('Deploy to Dev') {
            steps {
                script {
                    deployToenvironment('dev')
                }
            }
        }

        stage('Deploy to QA') {
            steps {
                script {
                    deployToenvironment('qa')
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    deployToenvironment('staging')
                }
            }
        }

        stage('Deploy to Prod') {
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                script {
                    deployToenvironment('prod')
                }
            }
        }
    }
    post {
        failure {
            echo "This will run if the job failed"
            mail to: "alimoviee@gmail.com",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
                body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
        }
    }
}

def deployToenvironment(env) {
    withEnv(["NAMESPACE=${env}"]) {
        sh '''
            rm -Rf .kube
            mkdir .kube
            ls
            cat $KUBECONFIG > .kube/config                     
            sudo helm upgrade --install app helm/ \
                --values=helm/values.yaml \
                --namespace $NAMESPACE \
                --set movie_service.image.repository=$DOCKER_MOVIE_IMAGE \
                --set movie_service.image.tag=$DOCKER_TAG \
                --set cast_service.image.repository=$DOCKER_CAST_IMAGE \
                --set cast_service.image.tag=$DOCKER_TAG \
                --set movie_db.secret.db=$MOVIE_DB_NAME
                --set movie_db.secret.username=$MOVIE_DB_USER
                --set movie_db.secret.password=$MOVIE_DB_PASSWORD
                --set cast_db.secret.db=$CAST_DB_NAME
                --set cast_db.secret.username=$CAST_DB_USER
                --set cast_db.secret.password=$CAST_DB_PASSWORD
            '''
    }
}