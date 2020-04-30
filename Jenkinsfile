pipeline {
    agent any
    environment {
        registry = "2017330017/spring-web"
        args '-v /var/run/docker.sock:/var/run/docker.sock'
    }
    stages {
        stage('Build') {
            agent {
                docker {
                     image 'maven:3-alpine' 
                     args '-v /root/.m2:/root/.m2' 
                }
            }
            steps {
                sh 'Hello World'
            }
        }
    }
}