pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = "reddit-clone-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "shubhz1452"
        DOCKER_PASS = "dockerhub-cred"
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }
    stages {
        stage("clean workspace") {
            steps {
                cleanWs()
            }
        }
        
        stage("Checkout from Git") {
            steps {
                git credentialsId: 'github-cred', url: 'https://github.com/shubhamk-tech/reddit-clone.git'
            }
        }
        
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=reddit-clone \
                    -Dsonar.projectKey=reddit-clone '''
                }
            }
        }
        
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage("Install Dependencies") {
            steps {
                sh "npm install"
            }
        }
        
        stage("Trivy Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        
        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }
        
        stage("Trivy Image Scan") {
            steps {
                script {
                    sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image shubhz1452/reddit-clone-pipeline:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table > trivyimage.txt')
                }
            }
        }
        
        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user cloud:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-15-206-193-14.ap-south-1.compute.amazonaws.com:8080/job/reddit-clone-CD/buildWithParameters?token=gitops-token'"
                }
            }
        }
    }
    
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                    "Build Number: ${env.BUILD_NUMBER}<br/>" +
                     "URL: ${env.BUILD_URL}<br/>",
                to: 'shubhkal01@gmail.com',                              
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
