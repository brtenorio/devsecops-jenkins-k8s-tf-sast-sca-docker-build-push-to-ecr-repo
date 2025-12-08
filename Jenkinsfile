def app
def registry = '555977951959.dkr.ecr.us-east-1.amazonaws.com'
def repo = 'flask-app'
def tag  = 'latest'

pipeline {
  agent any
  environment {
    AWS_REGION = 'us-east-1'
    DEPLOY = 'true'
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
        expression { return env.DEPLOY == 'true' }
      }
      steps {
        script {
          def image = "${registry}/${repo}:${tag}"
          // use stored kubeconfig credential (credentialId: 'kubelogin') to run kubectl
          // ensure 'kubelogin' is a File credential containing kubeconfig
          withCredentials([file(credentialsId: 'kubelogin', variable: 'KUBECONFIG')]) {
            // avoid Groovy interpolation of the secret by using single-quoted script strings
            sh 'kubectl --kubeconfig=$KUBECONFIG apply -f deployment.yaml -n flaskapp'
            sh 'kubectl --kubeconfig=$KUBECONFIG set image deployment/flask-app-deployment flask-app=' + image + ' -n flaskapp'
            try {
              sh 'kubectl --kubeconfig=$KUBECONFIG rollout status deployment/flask-app-deployment --timeout=900s -n flaskapp'
            } catch (err) {
              // collect useful debug info (pods, describe, logs) without revealing secret in logs
              sh 'echo "=== PODS ==="; kubectl --kubeconfig=$KUBECONFIG get pods -n flaskapp -o wide || true'
              sh 'echo "=== EVENTS ==="; kubectl --kubeconfig=$KUBECONFIG get events -n flaskapp --sort-by=.metadata.creationTimestamp || true'
              sh 'echo "=== DESCRIBE PODS ==="; for p in $(kubectl --kubeconfig=$KUBECONFIG get pods -n flaskapp -o name); do kubectl --kubeconfig=$KUBECONFIG describe $p -n flaskapp || true; done'
              sh 'echo "=== CONTAINER LOGS (first pod) ==="; POD=$(kubectl --kubeconfig=$KUBECONFIG get pods -n flaskapp -o jsonpath="{.items[0].metadata.name}"); kubectl --kubeconfig=$KUBECONFIG logs -n flaskapp $POD || true'
              error "Deployment rollout failed: ${err}"
            }
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