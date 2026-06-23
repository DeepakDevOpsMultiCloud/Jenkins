pipeline {
    agent any

    parameters {
        string(name: 'BRANCH', description: 'Insert the branch or commit-id')
        string(name: 'VERSION', description: 'Docker image tag')
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Deploy after build?')
    }

    environment {
        SERVICE_NAME   = 'admin-new-ui'
        AWS_REGION     = 'ap-south-1'
        AWS_PROFILE    = 'test-automation'
        ECR_ACCOUNT_ID = '851725619038'
        ECR_REPO_NAME  = 'ecr-uat'
        REMOTE_USER    = 'ubuntu'
        REMOTE_SERVER  = '43.204.145.52'
        REMOTE_DIR     = '/home/ubuntu/docker-apps/'
        IMAGE_NAME     = "${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}-${SERVICE_NAME}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scmGit(
                    branches: [[name: "${params.BRANCH}"]],
                    userRemoteConfigs: [[
                        credentialsId: 'Iu-jenkins-git-user',
                        url: 'https://github.com/infinite-uptime/admin-ui-v2.git'
                    ]]
                )
            }
        }

         stage('Pull shared scripts') {
            steps {
                withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        TOOLING_CLONE_URL=https://$GITHUB_TOKEN@github.com/infinite-uptime/ui-scripts.git \
                        sh scripts/load-scripts.sh
                    '''
                }
            }
        }

         stage('Sync env from ui-config') {
            steps {
                withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        GITHUB_TOKEN=$GITHUB_TOKEN sh scripts/remote/load-env.sh admin uat
                    '''
                }
            }
        }

        stage('Remove Old Local Image') {
            steps {
                sh """
                echo "Checking for old image..."
                docker rmi -f \$(docker images -q uat-${SERVICE_NAME}) || true
                """
            }
        }

         stage('Docker Build') {
            steps {
                withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_TOKEN')]) {
                    script {
                        echo "Building image for service: ${SERVICE_NAME}"
                        sh """
                            docker build --no-cache \\
                              -f Dockerfile \\
                              -t "${SERVICE_NAME}" \\
                              --build-arg GITHUB_TOKEN=\$GITHUB_TOKEN \\
                              .
                        """
                        echo "Image built successfully for ${SERVICE_NAME}"
                    }
                }
            }
        }

       stage('Push Image to ECR') {
            steps {
                sh """
                    echo "Logging into ECR..."
                    aws ecr get-login-password \
                        --region ${AWS_REGION} \
                        --profile ${AWS_PROFILE} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    echo "Tagging image..."
                    docker tag ${SERVICE_NAME} ${IMAGE_NAME}:${params.VERSION}
                    echo "Pushing image..."
                    docker push ${IMAGE_NAME}:${params.VERSION}
                """
            }
        }
        stage('Deploy to Remote Server') {
            when {
                expression { params.DEPLOY }
            }

            steps {

                sshagent(credentials: ['43.204.145.52']) {

                    sh """
                    ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_SERVER} '

                    cd ${REMOTE_DIR}${SERVICE_NAME} &&

                    echo "Updating IMAGE_TAG in .env file..." &&
                    sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=${params.VERSION}/" .env &&

                    echo "Logging into ECR..." &&
                    sudo aws ecr get-login-password \
                        --region ${AWS_REGION} \
                    | sudo docker login --username AWS \
                        --password-stdin ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com &&

                    echo "Pulling latest image..." &&
                    sudo docker compose pull &&

                    echo "Restarting container..." &&
                    sudo docker compose down &&
                    sudo docker compose up -d
                    '
                    """
                }
            }
        }
    }
}