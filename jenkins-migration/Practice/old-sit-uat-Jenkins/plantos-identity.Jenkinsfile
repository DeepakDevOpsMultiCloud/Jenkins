pipeline {
    agent any
    parameters {
        string description: 'Input the branch name or commit-id', name: 'BRANCH'
    }
    
    environment {
        AWS_REGION = 'ap-south-1'
        ECR_ACCOUNT_ID = '729397435375'
        ECS_CLUSTER_NAME = 'IU-UAT-NEW'
        ECR_REPO_NAME = "ecr-uat-new"
        TASK_DEFINITION_FAMILY = "td-uat-new-"
        serviceName = 'plantos-identity-api'
        DELAY = '15' // Delay in seconds
        MAX_ATTEMPTS = '30' // Max attempts
    }
    
    tools {
        jdk 'java_21'
        maven 'maven_3.8.8'
    }
    
    stages {
        
        stage('Set Build Description') {
            steps {
                script {
                    currentBuild.description = "#${env.BUILD_NUMBER}, Branch: ${params.BRANCH}, Version: ${params.VERSION}, ServicesBuilt: ${params.SERVICES_TO_BUILD}, ServicesDeployed: ${params.DEPLOY}"
                }
            }
        }
        
        stage('Workspace Cleanup') {
            when {
                expression { params.BUILD_STAGE == 'Build' }
            }
            steps {
                cleanWs()
                // Other build steps...
            }
        }
        
        stage('Pull Code From Github Onto Jenkins Server') {
            when {
                expression { params.BUILD_STAGE == 'Build' }
            }
            steps {
                checkout scmGit(branches: [[name: params.BRANCH]], extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: '/opt/uat-new/plantos-identity-api']], userRemoteConfigs: [[credentialsId: 'prem', url: 'https://github.com/infinite-uptime/plantos-identity-api.git']])
                // Change ownership of the entire directory
                    sh 'sudo chown -R jenkins:jenkins /opt/uat-new/plantos-identity-api'
            }
        }
        
        stage('Maven Build Jar Creation') {
            when {
                expression { params.BUILD_STAGE == 'Build' }
            }
            steps {
                script {
                echo "Building maven jar for service :- ${serviceName}..."
                sh """cd /opt/uat-new/${serviceName}
                mvn clean install -Dmaven.test.skip=true
                """
        }
            }
    }
    
        stage('Delete old image from the jenkins server') {
            when {
                expression { params.BUILD_STAGE == 'Build' }
            }
            steps {
                script {
                echo 'Remove old images from the jenkins server'
                // xargs docker rmi -f: Uses xargs to take the output (image IDs) from the previous awk command and pass them as arguments to the docker rmi -f command. 
                // -f is an option for docker rmi that forces the removal of the image.
                
                echo "Checking if any old image exists on jenkins server"
                def imageExists = sh(script: 'docker images | grep "$serviceName"', returnStatus: true) == 0

                if (imageExists) {
                echo "Image found, deleting..."
                sh 'docker images --format "{{.ID}} {{.Repository}}" | grep "$serviceName" | awk \'{print $1}\' | xargs -r docker rmi -f'
                } else {
                echo "Image not found, continuing to the next stage..."
                }
        }
            }
    }
    
        stage('Docker Build') {
            when {
                expression { params.BUILD_STAGE == 'Build' }
            }
            steps {
                echo "building image for service :- ${serviceName}"
                sh """cd /opt/uat-new/${serviceName}
                docker build -t ${serviceName} ."""
                echo "Image built successfully"
            }
    }
    
        stage('Push Newly Created Image To ECR') {
            when {
                expression { params.BUILD_STAGE == 'Build' }
            }
            steps {
                script {
                echo 'ECR login'
                //no userid, password for ecr login as instance has role with ecr full access
                sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com'
                echo "Pushing image for service :${serviceName} with tags version and latest"
                //service = "${serviceName}"
                //echo "Service : ${service}"
                
                sh """docker image tag \"$serviceName\" ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/\"$ECR_REPO_NAME\"-"${serviceName}\":latest
                docker push ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/\"$ECR_REPO_NAME\"-"${serviceName}\":latest

                docker image tag \"$serviceName\" ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/\"$ECR_REPO_NAME\"-"${serviceName}\":\"${VERSION}\"
                docker push ${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/\"$ECR_REPO_NAME\"-"${serviceName}\":\"$VERSION\""""
                
           def repositoryUri = sh(script: "aws ecr describe-repositories --repository-name ${ECR_REPO_NAME}-${serviceName} --region ${AWS_REGION} --query 'repositories[0].repositoryUri' --output text", returnStdout: true).trim()
           echo "repositoryUri : ${repositoryUri}"
           ECR_URI = "${repositoryUri}:${VERSION}"
           echo "URI of the latest image: ${ECR_URI}"

            }
            }
    }
    
        stage('Update ECS Task Definition with current running TD') {
            when {
                expression { params.DEPLOY != '' || params.BUILD_STAGE == '' }
            }
            steps {
                script {
                    
                    // Command to retrieve the current running task definition ARN
                    def taskDefinitionArn = sh(
                        script: "aws ecs describe-services --cluster ${ECS_CLUSTER_NAME} --services ecs-uat-new-${serviceName} --query 'services[0].taskDefinition' --output text",
                        returnStdout: true
                    ).trim()

                    // Extract revision number from the current running task definition ARN using AWK
                    def revisionNumber = sh(
                        script: "echo '${taskDefinitionArn}' | awk -F':' '{print \$NF}'",
                        returnStdout: true
                    ).trim()

                    // Output task definition ARN and revision number
                    echo "Task Definition ARN: ${taskDefinitionArn}"
                    echo "Revision Number: ${revisionNumber}"
                    
                    def TASK_DEFINITION = sh(script: "aws ecs describe-task-definition --task-definition \"${taskDefinitionArn}\" --region ${AWS_REGION}", returnStdout: true).trim()
                    echo "TASK_DEFINITION : ${TASK_DEFINITION}"

                    def NEW_TASK_DEFINITION = sh(script: """
    echo '$TASK_DEFINITION' | jq --arg IMAGE '$ECR_URI' '.taskDefinition | .containerDefinitions[0].image = \$IMAGE | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.compatibilities) | del(.registeredAt) | del(.registeredBy) '
""", returnStdout: true).trim()

                    echo "NEW_TASK_DEFINITION : ${NEW_TASK_DEFINITION}"

                    sh "aws ecs register-task-definition --region ${AWS_REGION} --cli-input-json '$NEW_TASK_DEFINITION'"
                    
                }
            }
        }
    
     stage('Deploy ECS service using the latest task definition created in above step ') {
         when {
                expression { params.DEPLOY != '' || params.BUILD_STAGE == '' }
            }
             steps {
                 script {
                 echo "Deploying service :${DEPLOY}"
                
                echo "Deploying ${DEPLOY} servcie : "
                        
                taskRevision = sh(script: """aws ecs describe-task-definition --task-definition td-uat-new-${DEPLOY} | egrep "revision" | tr "/" " " | awk '{print \$2}' | sed 's/,//'""", returnStdout: true).trim()
                   
                        sh """aws ecs update-service \
                        --cluster ${ECS_CLUSTER_NAME} \
                        --service ecs-uat-new-"$DEPLOY" \
                        --task-definition td-uat-new-${DEPLOY}:${taskRevision} \
                        --region ${AWS_REGION} 
                        """
             }
     }
}

        stage('Check ECS Service Stability') {
            when {
                expression { params.DEPLOY != '' || params.BUILD_STAGE == '' }
            }
            steps {
                script {
                    
                    //sh 'cd /opt/uat-new/drs/Backend-Services'
                    // Run the Python script to wait for the ECS service to stabilize
                    echo 'Waiting for the ECS service to stabilize...'
                    try {
                        sh """
                        python3 /opt/service-stable.py \
                            ${ECS_CLUSTER_NAME} \
                            ecs-uat-new-${serviceName} \
                            ${AWS_REGION} \
                            ${DELAY} \
                            ${MAX_ATTEMPTS}
                        """
                        echo 'ECS service is stable.'
                    } catch (Exception e) {
                        error 'Timeout reached while waiting for ECS service stability.Service has been deployed successfully but it has not reached stable state since 7 minutes. Please check the ECS events section for further details for unstability'
                    }
                }
            }
        }

}

}