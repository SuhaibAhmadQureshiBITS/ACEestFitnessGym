environment {
    AWS_REGION        = 'us-east-1'
    ECR_REPO          = '050916357370.dkr.ecr.ap-south-1.amazonaws.com/aceest-fitness'
    IMAGE_TAG         = "${env.BUILD_NUMBER}"
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
}
pipeline {
    agent any

    environment {
        IMAGE_NAME  = "aceest-fitness"
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        LOCAL_BIN   = "/root/.local/bin"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Source code checked out from GitHub"
            }
        }

        stage('Build, Lint & Test') {
            agent {
                docker {
                    image 'python:3.11'
                    args '-u root'   // needed for pip install
                }
            }
            steps {
                sh 'pip install --break-system-packages -r requirements.txt'
                sh '${LOCAL_BIN}/flake8 app.py test_app.py --max-line-length=100 || true'
                sh 'python -m pytest test_app.py -v --tb=short'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                echo "Docker image built: ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
	stage('Docker Build') {
 	   steps {
 	       sh """
       		     aws ecr get-login-password --region ${AWS_REGION} | \
       		     docker login --username AWS --password-stdin ${ECR_REPO}
        	     docker build -t ${ECR_REPO}:${IMAGE_TAG} .
          	     docker push ${ECR_REPO}:${IMAGE_TAG}
     		   """
   		}
	}
	stage('Deploy to EKS') {
    	   steps {
     	       sh """
    		      aws eks update-kubeconfig --region ${AWS_REGION} --name aceest-cluster
         	      sed -i 's|aceest-fitness:latest|${ECR_REPO}:${IMAGE_TAG}|g' k8s-rolling.yaml
                      kubectl apply -f k8s-rolling.yaml
                      kubectl rollout status deployment/aceest-rolling
                """
    		}
	}

        stage('Quality Gate') {
            steps {
                sh """
                    docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} \
                        python3 -m pytest test_app.py -v --tb=short
                """
                echo "Containerized tests passed — Quality Gate cleared"
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS — Build #${env.BUILD_NUMBER} passed all stages"
        }
        failure {
            echo "Pipeline FAILED — Review logs for Build #${env.BUILD_NUMBER}"
        }
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}
