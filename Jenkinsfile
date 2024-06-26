pipeline {
    agent { label 'jenkinsslave' }
    environment {
        DOCKER_CREDENTIALS_ID = 'docker_hub_login'
        SSH_CREDENTIALS_ID = 'kubeconfig'
        DOCKER_IMAGE_NAME = "deeeye2/pipeline-train-schedule"
        JAVA_HOME = '/usr/lib/jvm/java-11-openjdk-11.0.23.0.9-2.el7_9.x86_64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: 'master'
                    echo "Checking out branch: ${branch}"
                    checkout([$class: 'GitSCM',
                        branches: [[name: "*/${branch}"]],
                        userRemoteConfigs: [[url: 'https://github.com/deeeye2/cicd-pipeline-train-schedule-autodeploy.git']]
                    ])
                    echo "Checked out branch: ${branch}"
                }
            }
        }
        stage('Print Branch Info') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: 'master'
                    echo "Branch name: ${branch}"
                    sh 'env' // Print all environment variables for debugging
                }
            }
        }
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    def app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', DOCKER_CREDENTIALS_ID) {
                        def app = docker.image(DOCKER_IMAGE_NAME)
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                script {
                    withCredentials([file(credentialsId: SSH_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
                        sh 'kubectl apply -f train-schedule-kube-canary.yml --kubeconfig=$KUBECONFIG'
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                script {
                    withCredentials([file(credentialsId: SSH_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
                        sh 'kubectl apply -f train-schedule-kube-canary.yml --kubeconfig=$KUBECONFIG'
                        sh 'kubectl apply -f train-schedule-kube.yml --kubeconfig=$KUBECONFIG'
                    }
                }
            }
        }
    }
}

