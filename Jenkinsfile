pipeline {
  agent any

  environment {
    ECR_REG = "303927185771.dkr.ecr.us-east-1.amazonaws.com" // replace or set via Jenkins credentials/env
    IMAGE = "${env.ECR_REG}/devops-ci-cd-demo-app:${env.GIT_COMMIT}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Docker Image') {
      steps {
        sh 'docker --version'
        sh "docker build -t ${IMAGE} ."
      }
    }

    
    stage('Login to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']]) {
          sh '''
            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_REG}
          '''
        }
      }
    }





    stage('Push Image') {
      steps {
        script {
          def imageDigest = sh(
            script: "docker push ${IMAGE} --quiet",
            returnStdout: true
          ).trim()
          echo "✅ Image pushed successfully. Digest: ${imageDigest}"
        }
      }
    }


    stage('Deploy via Ansible') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'app-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          sh '''
            # prepare inventory dynamically or use the repo inventory
            ansible-playbook -i ansible/inventory.ini ansible/deploy.yml -e "image_tag=${GIT_COMMIT}" -e "ECR_REG=${ECR_REG}"
          '''
        }
      }
    }
  }
}













