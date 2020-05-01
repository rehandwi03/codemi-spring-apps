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
                sh 'mvn clean package' 
            }
        }
        stage('Build') {
            environment {
                registryCredential = 'dockerhub'
            }
            steps {
                sh 'cd ${WORKSPACE}/'
                script {
                    def appimage = docker.build registry + ":$BUILD_NUMBER"
                    docker.withRegistry('', registryCredential) {
                        appimage.push()
                        appimage.push('latest')
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    def image_id = registry + ":$BUILD_NUMBER"
                    sh "ansible-playbook playbook.yml --extra-vars \"image_id=${image_id}\""
                }
            }
        }
    }
}