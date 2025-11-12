pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('e22f124e-2767-44cf-930d-23dd19c842a7')
        GITHUB_CREDENTIALS = credentials('da270b62-31a2-44ed-a570-c701a933abf6')
        DOCKER_IMAGE = '66raven99/java-web-app:latest'
        K8S_NAMESPACE = 'default'
        K8S_DEPLOYMENT = 'java-web-app'
    }

    stages {
        stage('Build Maven Project') {
            steps {
                sh './mvnw clean install'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $DOCKER_IMAGE ."
            }
        }

        stage('Login to DockerHub') {
            steps {
                sh "echo $DOCKERHUB_CREDENTIALS_PSW | docker login --username $DOCKERHUB_CREDENTIALS_USR --password-stdin"
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push $DOCKER_IMAGE"
            }
        }

        stage('Deploy to Kubernetes') {
           steps {
        withCredentials([file(credentialsId: 'kubeconfig-minikube', variable: 'KUBECONFIG')]) {
            sh """
                echo "Using kubeconfig from Jenkins secret file"
                kubectl --kubeconfig="$KUBECONFIG" config get-contexts
                kubectl --kubeconfig="$KUBECONFIG" set image deployment/$K8S_DEPLOYMENT $K8S_DEPLOYMENT=$DOCKER_IMAGE -n $K8S_NAMESPACE
                kubectl --kubeconfig="$KUBECONFIG" rollout status deployment/$K8S_DEPLOYMENT -n $K8S_NAMESPACE
            """
        }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs.'
        }
    }
}
