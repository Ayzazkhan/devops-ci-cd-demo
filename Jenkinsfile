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
          docker build -t ${FULL_IMAGE} .
        """
      }
    }

    stage('Login to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
          sh """
            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REG}
          """
        }
      }
    }

    stage('Push Image') {
      steps {
        sh "docker push ${FULL_IMAGE}"
      }
    }

    stage('Setup SSH') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'app-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          sh """
            # Setup SSH directory and known_hosts
            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            ssh-keyscan -H ${TARGET_HOST} >> ~/.ssh/known_hosts
            chmod 600 ~/.ssh/known_hosts
            
            # Test SSH connection
            ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no ec2-user@${TARGET_HOST} 'echo "SSH connection successful"'
          """
        }
      }
    }

    stage('Deploy via Ansible') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'app-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          sh """
            # Copy SSH key to current directory for Ansible
            cp ${SSH_KEY} ./ssh_key.pem
            chmod 600 ./ssh_key.pem
            
            # Deploy with Ansible
            ansible-playbook -i ansible/inventory.ini ansible/deploy.yml \
              -e "image_tag=${IMAGE_TAG}" \
              -e "ECR_REG=${ECR_REG}" \
              --private-key=./ssh_key.pem \
              -e "ansible_ssh_user=ec2-user"
          """
        }
      }
    }
  }

  post {
    always {
      sh """
        # Cleanup SSH key
        rm -f ./ssh_key.pem || true
        docker rmi ${FULL_IMAGE} || true
      """
    }
  }
}
