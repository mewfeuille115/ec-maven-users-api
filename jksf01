pipeline {
    // agent { label 'controller' }
    agent any
    environment {
        IMAGE_NAME = "mi-aplicacion-java"
        IMAGE_TAG = "latest"
        DOCKERFILE_PATH = "Dockerfile"
        // DOCKER_CREDS = credentials('azcontainerregistrycna')
        ACR_REGISTRY = "azcontainerregistrycna.azurecr.io"
        APP_NAME = "myapp"
     }
 

    stages {
        stage('Compilar con Maven') {
            agent {
                docker { image 'maven:3.8.4-openjdk-17-slim' }
            }
            steps {
                sh 'mvn clean package'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: true
            }
        }

        // stage('Sonarqube') {
        //     agent {
        //         docker { image 'maven:3.8.4-openjdk-17-slim' }
        //     }
        //     steps {

        //         withSonarQubeEnv('sonar-server') {
        //             sh 'mvn clean package org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
        //                     -Dsonar.projectKey=labmaven01 \
        //                     -Dsonar.projectName=labmaven01 \
        //                     -Dsonar.sources=src/main \
        //                     -Dsonar.sourceEncoding=UTF-8 \
        //                     -Dsonar.language=java \
        //                     -Dsonar.tests=src/test \
        //                     -Dsonar.junit.reportsPath=target/surefire-reports \
        //                     -Dsonar.surefire.reportsPath=target/surefire-reports \
        //                     -Dsonar.jacoco.reportPath=target/jacoco.exec \
        //                     -Dsonar.java.binaries=target/classes \
        //                     -Dsonar.java.coveragePlugin=jacoco \
        //                     -Dsonar.coverage.jacoco.xmlReportPaths=target/jacoco.xml \
        //                     -Dsonar.exclusions=**/*IT.java,**/*TEST.java,**/*Test.java,**/src/it**,**/src/test**,**/gradle/wrapper** \
        //                     -Dsonar.java.libraries=target/*.jar'
        //         }
        //     }
        // }

        stage('Build Image') {
             steps {
                 copyArtifacts filter: 'target/*.jar',
                               fingerprintArtifacts: true,
                               projectName: '${JOB_NAME}',
                               flatten: true,
                               selector: specific('${BUILD_NUMBER}'),
                               target: 'target'
                 sh "docker build -t ${ACR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -f ${DOCKERFILE_PATH} ."
             }
         }
 
        stage('Publish Image') {
            steps {
                script {
                    //sh 'docker login ${ACR_REGISTRY} -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}'
                    //sh 'docker tag ${ACR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${ACR_REGISTRY}/${IMAGE_NAME}:$BUILD_NUMBER'
                    //sh 'docker push ${ACR_REGISTRY}/${IMAGE_NAME}:$BUILD_NUMBER'
                    sh 'docker logout'
                }
            }
        }

        // stage('Deploy') {
        //     agent {
        //         docker { image 'mcr.microsoft.com/azure-cli' }
        //     }
        //     steps {
        //         script {
        //             sh """az containerapp up \
        //                     --name ${APP_NAME} \
        //                     --image ${ACR_REGISTRY}/${IMAGE_NAME}:$BUILD_NUMBER \
        //                     --ingress external \
        //                     --target-port 8080"""
        //         }
        //     }
        // }

    }
    post {
        success {
            echo 'La compilación y las pruebas fueron exitosas.'
        }
        failure {
            echo 'Hubo un error en la compilación o las pruebas.'
        }
    }
}




