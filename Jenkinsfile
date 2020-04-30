pipeline {
    agent {
        docker {
            image 'adoptopenjdk/openjdk11:alpine-jre'
        }
    }
    environment {
        registry = "2017330017/spring-web"
    }
    stages {
        stage('Build') { 
            steps {
                sh 'mvn -B -DskipTests clean package' 
            }
        }
    }
}