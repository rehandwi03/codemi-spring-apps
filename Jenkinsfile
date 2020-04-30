pipeline{
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
                sh 'mvn clean package'
            }
        }
        stage('Publish') {
        //     environment {
        //         registryCredential = 'dockerhub'
        //     }
        //     steps {
        //         script {
        //             def appimage = docker.build registry + ":$BUILD_NUMBER"
        //             docker.withRegistry('', registryCredential) {
        //                 appimage.push()
        //                 appimage.push('latest')
        //             }
        //         }
        //     }
        // }
        // stage('Deploy') {
        //     steps {
        //         script {
        //             def image_id = registry + ":$BUILD_NUMBER"
        //             sh "ansible-playbook playbook.yml --extra-vars \"image_id=${image_id}\""
        //         }
        //     }
        // }
    }
}