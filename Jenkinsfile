pipeline {
    agent {
        docker {
            image 'maven:3.6.3-jdk-11' 
            args '-v /root/.m2:/root/.m2' 
        }
    }
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
                sh 'mvn -B -DskipTests clean package' 
            }
        }
        stage('Publish') {
            environment {
                registryCredential = 'dockerhub'
            }
            steps {
                script {
                    def appimage = docker.build registry + ":$BUILD_NUMBER"
                    docker.withRegistry('', registryCredential) {
                        appimage.push()
                        appimage.push('latest')
                    }
                }
          
    }
}