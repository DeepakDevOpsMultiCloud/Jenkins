pipeline {
    agent {
        label "linux-node"
    }

    environment {
        deploy_env 
    }

    parameters {

    }

    stages {

        stage(checkout) {
            steps {
                   echo "git checkout URL"
            }


        }
        stage(build) {
            steps {

                  echo "building code"
            }


        }
        stage(test){
            steps {
                 echo "testing"
            }


        }
        stage(docker image build){
            steps {
              echo "docker image build -t"
            }
        }

        stage(image push) {
            steps {
                echo "docker image push ecr repo"
            }

        }

        stage(deploy) {
            steps {
                echo "update service with new td"
            }
        }

    }

}