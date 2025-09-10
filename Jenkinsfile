pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', daysToKeepStr: '5'))
        timestamps()
    }

    environment {
        registry = 'xavielee/house-price-prediction-api'
        registryCredential = 'dockerhub'
        APP_DIR = 'lesson11_CICD_jenkins'
    }

    stages {
        stage('Test') {
            agent {
                docker { image 'python:3.8' }
            }
            steps {
                echo 'Testing model correctness..'
                dir(env.APP_DIR) {
                    sh 'pip install -r requirements.txt && pytest'
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    echo 'Building image for deployment..'
                    def imageTag = "${registry}:${env.BUILD_NUMBER}"
                    // Build with the Dockerfile inside the app directory
                    dockerImage = docker.build(imageTag, "-f ${env.APP_DIR}/Dockerfile ${env.APP_DIR}")
                    echo 'Pushing image to dockerhub..'
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying models..'
                echo 'Trigger remote deployment script or compose here.'
            }
        }
    }
}

