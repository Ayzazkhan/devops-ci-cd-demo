pipeline {
  agent any

  environment {
    ECR_REG = "303927185771.dkr.ecr.us-east-1.amazonaws.com"
    IMAGE_NAME = "devops-ci-cd-demo-app"
    IMAGE_TAG = "${env.GIT_COMMIT ?: 'latest'}"
    FULL_IMAGE = "${env.ECR_REG}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
    TARGET_HOST = "54.172.91.142"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          echo "Building Docker image: ${FULL_IMAGE}"
          docker build -t ${FULL_IMAGE} .
        """
      }
    }

    stage('Login to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
          sh """
            echo "Logging into ECR..."
            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REG}
          """
        }
      }
    }

    stage('Push Image') {
      steps {
        sh """
          echo "Pushing image to ECR..."
          docker push ${FULL_IMAGE}
          echo "✅ Image pushed successfully!"
        """
      }
    }

    stage('Setup SSH') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'app-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          sh """
            echo "Setting up SSH configuration..."
            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            ssh-keyscan -H ${TARGET_HOST} >> ~/.ssh/known_hosts 2>/dev/null
            chmod 600 ~/.ssh/known_hosts
          """
        }
      }
    }

    stage('Deploy via Ansible') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'app-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          sh """
            echo "Starting Ansible deployment..."
            # Copy and secure SSH key
            cp ${SSH_KEY} ./ssh_key.pem
            chmod 600 ./ssh_key.pem
            
            # Set Ansible environment variables
            export ANSIBLE_HOST_KEY_CHECKING=False
            export ANSIBLE_SSH_RETRIES=3
            
            # Run Ansible deployment
            ansible-playbook -i ansible/inventory.ini ansible/deploy.yml \
              -e "image_tag=${IMAGE_TAG}" \
              -e "ECR_REG=${ECR_REG}" \
              --private-key=./ssh_key.pem \
              -e "ansible_ssh_user=ec2-user" \
              -v
          """
        }
      }
    }
  }

  post {
    always {
      script {
        echo "Cleaning up resources..."
        sh """
          # Remove SSH key safely
          rm -f ./ssh_key.pem 2>/dev/null || true
          
          # Remove Docker image if it exists
          if docker images --format "table {{.Repository}}:{{.Tag}}" | grep -q "${FULL_IMAGE}"; then
            docker rmi ${FULL_IMAGE} 2>/dev/null || echo "Image already removed"
          else
            echo "Docker image not found, skipping removal"
          fi
        """
      }
    }
    
    success {
      echo "✅ Pipeline completed successfully!"
      sh """
        echo "Deployment completed for image: ${FULL_IMAGE}"
      """
    }
    
    failure {
      echo "❌ Pipeline failed!"
      // Add notification here if needed
    }
  }
}
