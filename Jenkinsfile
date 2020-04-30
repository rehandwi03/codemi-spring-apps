pipeline {
    agent any
    environment {
        registry = "2017330017/spring-web"
    }
    stages {
        stage('Clean') { 
            agent {
                docker {
                    image 'maven:3.6.3-jdk-11' 
                    args '-v /root/.m2:/root/.m2' 
                    }
            }
            steps {
                sh 'mvn -B -DskipTests clean package' 
            }
        }
    }
}