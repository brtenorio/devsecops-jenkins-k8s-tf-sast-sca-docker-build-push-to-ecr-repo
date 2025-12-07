def app
def registry = '555977951959.dkr.ecr.us-east-1.amazonaws.com'
def repo = 'flask-app'
def tag  = 'latest'

pipeline {
  agent any
  environment {
    AWS_REGION = 'us-east-1'
  }

  stages {
    stage('Build') {
      steps {
        script {
          // build image from the flask-app/ directory and tag with full ECR name
          app = docker.build("${registry}/${repo}:${tag}", "flask-app/")
        }
      }
    }

    stage('Push') {
      steps {
        script {
          // Recommended: login to ECR with AWS CLI on the agent
          sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${registry}"
          app.push()

          // Alternative (if you have a working Jenkins credential for ECR):
          // docker.withRegistry("https://${registry}", 'ecr:us-east-1:aws-credentials') {
          //   app.push()
          // }
        }
      }
    }

    stage('Deploy') {
      when {
        expression { return env.DEPLOY == 'true' } // optional gate; set DEPLOY=true to deploy
      }
      steps {
        script {
          // prerequisites on the agent: aws-cli, kubectl, and IAM user/role able to access the EKS cluster
          // configure kubeconfig for the EKS cluster (cluster name must match your eksctl create cluster)
          sh "aws eks update-kubeconfig --name kubernetes-cluster --region ${env.AWS_REGION}"

          // update the deployment image to the newly pushed image and wait for rollout
          sh """
            kubectl set image deployment/flask-app-deployment flask-app=${registry}/${repo}:${tag} --record
            kubectl rollout status deployment/flask-app-deployment --timeout=120s
          """
        }
      }
    }
  }

  post {
    always {
      script {
        // cleanup optional
        sh 'docker image prune -f || true'
      }
    }
  }
}