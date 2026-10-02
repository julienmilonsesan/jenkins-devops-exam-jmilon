pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USER = 'julienmilonsesan'
        IMAGE_MOVIE = "${DOCKERHUB_USER}/movie-service"
        IMAGE_CAST = "${DOCKERHUB_USER}/cast-service"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'echo "BRANCH_NAME=${BRANCH_NAME}"'
                sh 'git rev-parse --abbrev-ref HEAD'
            }
        }

        stage('Build Images') {
            steps {
                sh "docker build -t ${IMAGE_MOVIE}:${BUILD_NUMBER} -t ${IMAGE_MOVIE}:latest ./movie-service"
                sh "docker build -t ${IMAGE_CAST}:${BUILD_NUMBER} -t ${IMAGE_CAST}:latest ./cast-service"
            }
        }

        stage('Push Images') {
            steps {
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                sh "docker push ${IMAGE_MOVIE}:${BUILD_NUMBER}"
                sh "docker push ${IMAGE_MOVIE}:latest"
                sh "docker push ${IMAGE_CAST}:${BUILD_NUMBER}"
                sh "docker push ${IMAGE_CAST}:latest"
            }
        }

        stage('Deploy Dev') {
            steps {
                sh "kubectl apply -f k8s/movie-db.yaml -n dev"
                sh "kubectl apply -f k8s/cast-db.yaml -n dev"
                sh "kubectl apply -f k8s/nginx-configmap.yaml -n dev"
                sh "sed 's/nodePort: 30080/nodePort: 30080/' k8s/nginx.yaml | kubectl apply -f - -n dev"
                sh "helm upgrade --install movie-service charts -f charts/values.yaml -f charts/values-movie.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30001 -n dev"
                sh "helm upgrade --install cast-service charts -f charts/values.yaml -f charts/values-cast.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30002 -n dev"
            }
        }

        stage('Deploy QA') {
            steps {
                sh "kubectl apply -f k8s/movie-db.yaml -n qa"
                sh "kubectl apply -f k8s/cast-db.yaml -n qa"
                sh "kubectl apply -f k8s/nginx-configmap.yaml -n qa"
                sh "sed 's/nodePort: 30080/nodePort: 30081/' k8s/nginx.yaml | kubectl apply -f - -n qa"
                sh "helm upgrade --install movie-service charts -f charts/values.yaml -f charts/values-movie.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30011 -n qa"
                sh "helm upgrade --install cast-service charts -f charts/values.yaml -f charts/values-cast.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30012 -n qa"
            }
        }

        stage('Deploy Staging') {
            steps {
                sh "kubectl apply -f k8s/movie-db.yaml -n staging"
                sh "kubectl apply -f k8s/cast-db.yaml -n staging"
                sh "kubectl apply -f k8s/nginx-configmap.yaml -n staging"
                sh "sed 's/nodePort: 30080/nodePort: 30082/' k8s/nginx.yaml | kubectl apply -f - -n staging"
                sh "helm upgrade --install movie-service charts -f charts/values.yaml -f charts/values-movie.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30021 -n staging"
                sh "helm upgrade --install cast-service charts -f charts/values.yaml -f charts/values-cast.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30022 -n staging"
            }
        }

        stage('Deploy Prod') {
            when {
                branch 'main'
            }
            steps {
                input message: "Déployer en production ?", ok: "Déployer"
                sh "kubectl apply -f k8s/movie-db.yaml -n prod"
                sh "kubectl apply -f k8s/cast-db.yaml -n prod"
                sh "kubectl apply -f k8s/nginx-configmap.yaml -n prod"
                sh "sed 's/nodePort: 30080/nodePort: 30083/' k8s/nginx.yaml | kubectl apply -f - -n prod"
                sh "helm upgrade --install movie-service charts -f charts/values.yaml -f charts/values-movie.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30031 -n prod"
                sh "helm upgrade --install cast-service charts -f charts/values.yaml -f charts/values-cast.yaml --set image.tag=${BUILD_NUMBER} --set service.nodePort=30032 -n prod"
            }
        }
    }

    post {
        always {
            sh "docker logout"
        }
    }
}
