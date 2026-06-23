pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['Build', 'Deploy'],
            description: 'Choose whether to build (push image) or deploy'
        )

        string(
            name: 'COMMIT_SHA',
            description: 'Commit SHA (only for Deploy). For Build you may leave empty'
        )

        choice(
            name: 'ENVIRONMENT',
            choices: ['sit', 'uat', 'uat_new' ],
            description: 'Deployment environment (only for Deploy)'
        )

        string(
            name: 'BRANCH',
            defaultValue: 'develop',
            description: 'Branch name to checkout (only used during Build)'
        )
    }

    environment {
        REPO_URL = 'https://github.com/infinite-uptime/kpi-formula-validation-service.git'
        CHECKOUT_DIR = '/opt/sit/kpi-formula-validation-service'
        AWS_REGION = 'ap-south-1'
        AWS_PROFILE = 'sit-uat'
    }

    stages {

        /* ------------------------- SET DESCRIPTION ------------------------- */
        stage('Set Build Description') {
            steps {
                script {
                    currentBuild.description = "${params.ACTION} - ${params.COMMIT_SHA}"
                }
            }
        }

        /* ---------------------------- CLEAN WS ------------------------------ */
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        /* ------------------------------ CHECKOUT ---------------------------- */
        stage('Checkout Code') {
            when { expression { params.ACTION == 'Build' } }
            steps {
                checkout scmGit(
                    branches: [[name: "${params.BRANCH}"]],
                    extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: "${CHECKOUT_DIR}"]],
                    userRemoteConfigs: [[
                        credentialsId: 'Iu-jenkins-git-user',
                        url: "${REPO_URL}"
                    ]]
                )
                sh "sudo chown -R jenkins:jenkins ${CHECKOUT_DIR}"
            }
        }

        /* ------------------------------- BUILD ------------------------------ */
        stage('Build Image (CI)') {
            when { expression { params.ACTION == 'Build' } }
            steps {
                script {
                    sh """
                        cd ${CHECKOUT_DIR}
                        chmod +x infra/ci.sh
                        ./infra/ci.sh
                    """
                }
            }
        }

        /* ------------------------------- DEPLOY ----------------------------- */
        stage('Deploy (CD)') {
            when { expression { params.ACTION == 'Deploy' } }
            steps {
                script {
                    if (!params.COMMIT_SHA?.trim()) {
                        error("Commit SHA is required for deployment!")
                    }
                    sh """
                        cd ${CHECKOUT_DIR}
                        chmod +x infra/cd.sh
                        ./infra/cd.sh ${params.ENVIRONMENT} ${params.COMMIT_SHA}
                    """
                }
            }
        }
    }
}
