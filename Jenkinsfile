def app
def registry = '555977951959.dkr.ecr.us-east-1.amazonaws.com'
def repo = 'flask-app'
def tag  = 'latest'

pipeline {
  agent any

  parameters {
    booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Set true to deploy to k8s')
  }

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
          // login to ECR and push image
          sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${registry}"
          app.push()
        }
      }
    }

    stage('Deploy') {
      when {
        expression { return params.DEPLOY == true }
      }
      steps {
        script {
          def image = "${registry}/${repo}:${tag}"
          // use kubeconfig stored in Jenkins (File credential 'kubelogin')
          withCredentials([file(credentialsId: 'kubelogin', variable: 'KUBECONFIG')]) {
            // verify image exists in ECR
            sh 'if ! aws ecr describe-images --repository-name ' + repo + ' --image-ids imageTag=' + tag + ' --region ' + env.AWS_REGION + ' >/dev/null 2>&1; then echo "Image not found in ECR: ' + registry + '/' + repo + ':' + tag + '"; exit 1; fi'

            // prepare manifest (keep original file unchanged) and apply using kubeconfig
            // NOTE: this sed replaces port 5000 -> 5010 if your container listens on 5010.
            sh '''
              cp deployment.yaml deployment.apply.yaml || true
              sed -i.bak 's/containerPort: 5000/containerPort: 5010/g' deployment.apply.yaml || true
              sed -i.bak 's/port: 5000/port: 5010/g' deployment.apply.yaml || true
              sed -i.bak 's/targetPort: 5000/targetPort: 5010/g' deployment.apply.yaml || true
              kubectl --kubeconfig=$KUBECONFIG apply -f deployment.apply.yaml -n flaskapp
            '''

            // update image and wait for rollout (single-quoted sh avoids interpolating the kubeconfig secret)
            sh 'kubectl --kubeconfig=$KUBECONFIG set image deployment/flask-app-deployment flask-app=' + image + ' -n flaskapp'
            sh 'kubectl --kubeconfig=$KUBECONFIG rollout status deployment/flask-app-deployment --timeout=300s -n flaskapp'
          }
        }
      }
    }
  }

  post {
    always {
      script {
        sh 'docker image prune -f || true'
      }
    }
  }
}