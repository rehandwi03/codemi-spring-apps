pipeline {
    agent any
    environment {
        registry = "2017330017/spring-web"
    }
    stages {
        stage('Build') {
            agent {
                docker {
                    image 'adoptopenjdk/openjdk11:alpine-jre'
                }
            }
            steps {
                sh 'Hello World'
            }
        }
    }
}