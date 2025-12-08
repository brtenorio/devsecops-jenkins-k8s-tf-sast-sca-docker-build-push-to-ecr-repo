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
        }
      }
    }

    stage('Deploy') {
      when {
        expression { return env.DEPLOY == 'true' } // optional gate; set DEPLOY=true to deploy
      }
      steps {
        script {
          // use stored kubeconfig credential (credentialId: 'kubelogin') to run kubectl
          // ensure the Jenkins credential 'kubelogin' is of type "File" containing a kubeconfig
          withCredentials([file(credentialsId: 'kubelogin', variable: 'KUBECONFIG')]) {
            sh """
              kubectl --kubeconfig=${KUBECONFIG} apply -f deployment.yaml -n flaskapp
              kubectl --kubeconfig=${KUBECONFIG} set image deployment/flask-app-deployment flask-app=${registry}/${repo}:${tag} --record -n flaskapp
              kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/flask-app-deployment --timeout=120s -n flaskapp
            """
          }
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