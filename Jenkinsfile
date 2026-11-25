pipeline {
    agent any

    environment {
        PATH = "C:\\Program Files\\Docker\\Docker\\resources\\bin;${env.PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/aboubacar240031-pixel/MyAppReact.git'
            }
        }

        stage('Build images') {
            steps {
                bat 'docker build -t mern-server ./Server'
                bat 'docker build -t mern-client ./Client'
            }
        }

        stage('Deploy with Compose') {
            steps {
                bat 'docker compose down'
                bat 'docker compose up --build -d'
            }
        }

        stage('Test') {
            steps {
                bat 'ping -n 15 127.0.0.1 >nul'
                bat 'docker compose ps'
                bat 'curl -f http://localhost:3000'
            }
        }
    }

    post {
        failure {
            bat 'docker compose logs'
        }
    }
}