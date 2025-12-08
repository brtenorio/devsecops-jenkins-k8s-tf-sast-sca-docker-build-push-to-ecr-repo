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
            try {
              // ensure image exists in ECR before applying deployment
              sh 'if ! aws ecr describe-images --repository-name ' + repo + ' --image-ids imageTag=' + tag + ' --region ' + env.AWS_REGION + ' >/dev/null 2>&1; then echo "Image not found in ECR: ' + registry + '/' + repo + ':' + tag + '"; aws ecr list-images --repository-name ' + repo + ' --region ' + env.AWS_REGION + ' || true; exit 1; fi'
              // apply manifest and update image; avoid Groovy interpolation of KUBECONFIG by using single-quoted sh strings
              sh 'kubectl --kubeconfig=$KUBECONFIG apply -f deployment.yaml -n flaskapp'
              sh 'kubectl --kubeconfig=$KUBECONFIG set image deployment/flask-app-deployment flask-app=' + image + ' -n flaskapp'
              // wait for rollout (extended timeout)
              sh 'kubectl --kubeconfig=$KUBECONFIG rollout status deployment/flask-app-deployment --timeout=600s -n flaskapp'
            } catch (err) {
              // collect diagnostics without exposing the kubeconfig content
              sh 'echo "=== DEPLOYMENT ==="; kubectl --kubeconfig=$KUBECONFIG describe deployment/flask-app-deployment -n flaskapp || true'
              sh 'echo "=== PODS ==="; kubectl --kubeconfig=$KUBECONFIG get pods -n flaskapp -o wide || true'
              sh 'echo "=== EVENTS ==="; kubectl --kubeconfig=$KUBECONFIG get events -n flaskapp --sort-by=.metadata.creationTimestamp || true'
              sh 'echo "=== DESCRIBE PODS ==="; for p in $(kubectl --kubeconfig=$KUBECONFIG get pods -n flaskapp -o name); do kubectl --kubeconfig=$KUBECONFIG describe $p -n flaskapp || true; done'
              sh '''
                echo "=== CONTAINER LOGS (all pods) ==="
                for p in $(kubectl --kubeconfig=$KUBECONFIG get pods -n flaskapp -o jsonpath="{.items[*].metadata.name}"); do
                  echo "---- LOGS for $p ----"
                  kubectl --kubeconfig=$KUBECONFIG logs -n flaskapp $p --all-containers=true || true
                done
              '''
              sh 'echo "=== NODES ==="; kubectl --kubeconfig=$KUBECONFIG get nodes -o wide || true'
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