pipeline {
    agent any
    environment {
        registry = "2017330017/spring-web"
    }
    stages {
        agent {
            docker {
                image 'adoptopenjdk/openjdk11:alpine-jre'
            }
        }
        steps {
            echo 'Hello World'
        }
    }
}