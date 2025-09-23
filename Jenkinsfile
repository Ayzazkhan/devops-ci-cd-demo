pipeline {
  agent any

  environment {
    ECR_REG = "303927185771.dkr.ecr.us-east-1.amazonaws.com"
    IMAGE_NAME = "devops-ci-cd-demo-app"  // Fixed: Separate name from tag
    IMAGE_TAG = "${env.GIT_COMMIT ?: 'latest'}"  // Handle null GIT_COMMIT
    FULL_IMAGE = "${env.ECR_REG}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Docker Image') {
      steps {
        sh """
          docker --version
          echo "Building image: ${FULL_IMAGE}"
          docker build -t ${FULL_IMAGE} .
        """
      }
    }

    stage('Verify Image Exists') {
      steps {
        sh """
          echo "Checking built images:"
          docker images | grep "${IMAGE_NAME}" || echo "Image not found!"
          echo "Image details:"
          docker inspect ${FULL_IMAGE} || echo "Cannot inspect image"
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
        script {
          // Verify image exists before pushing
          sh """
            echo "Pushing image: ${FULL_IMAGE}"
            docker images | grep "${IMAGE_NAME}" || exit 1
          """
          
          // Push the image
          sh "docker push ${FULL_IMAGE}"
          
          // Get digest
          def imageDigest = sh(
            script: "docker inspect --format='{{index .RepoDigests 0}}' ${FULL_IMAGE}",
            returnStdout: true
          ).trim()
          echo "✅ Image pushed successfully: ${imageDigest}"
        }
      }
    }

    stage('Deploy via Ansible') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'app-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          sh """
            ansible-playbook -i ansible/inventory.ini ansible/deploy.yml -e "image_tag=${IMAGE_TAG}" -e "ECR_REG=${ECR_REG}"
          """
        }
      }
    }
  }

  post {
    always {
      sh """
        echo "Cleanup: Removing local Docker image"
        docker rmi ${FULL_IMAGE} || true
      """
    }
  }
}
