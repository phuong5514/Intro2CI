pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "intro2ci"
        DOCKER_REGISTRY = credentials('docker-hub-username')
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = bat(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                bat 'npm ci'
            }
        }
        
        stage('Run Tests') {
            steps {
                bat 'npm test'
            }
        }
        
        stage('Build Docker Image') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'main'
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                }
            }
            steps {
                script {
                    def imageTag = ""
                    def stage_name = ""
                    
                    if (env.BRANCH_NAME == 'dev') {
                        imageTag = "dev"
                        stage_name = "dev"
                    } else if (env.BRANCH_NAME == 'main') {
                        imageTag = "staging"
                        stage_name = "staging"
                    } else if (env.TAG_NAME) {
                        imageTag = "${env.TAG_NAME}"
                        stage_name = "production"
                    }
                    
                    env.IMAGE_TAG = imageTag
                    env.STAGE_NAME = stage_name
                    
                    bat """
                        docker build -t ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${imageTag} .
                    """
                    
                    if (env.TAG_NAME) {
                        bat """
                            docker tag ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${imageTag} \
                                ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:production
                        """
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'main'
                    tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
                }
            }
            steps {
                script {
                    bat """
                        echo ${DOCKER_CREDENTIALS_PSW} | docker login -u ${DOCKER_CREDENTIALS_USR} --password-stdin
                        docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${env.IMAGE_TAG}
                    """
                    
                    if (env.TAG_NAME) {
                        bat """
                            docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:production
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Development') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'render-deploy-hook-dev', variable: 'DEPLOY_HOOK')]) {
                        bat "curl -X POST ${DEPLOY_HOOK}"
                    }
                    echo 'Deployed to Development environment'
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'render-deploy-hook-staging', variable: 'DEPLOY_HOOK')]) {
                        bat "curl -X POST ${DEPLOY_HOOK}"
                    }
                    echo 'Deployed to Staging environment'
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                tag pattern: 'v\\d+\\.\\d+\\.\\d+', comparator: 'REGEXP'
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'render-deploy-hook-prod', variable: 'DEPLOY_HOOK')]) {
                        bat "curl -X POST ${DEPLOY_HOOK}"
                    }
                    echo 'Deployed to Production environment'
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully for ${env.STAGE_NAME} stage"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}

