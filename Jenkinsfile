pipeline {
  agent any
  tools { 
    maven 'Maven_3_5_2'  
  }
  stages {
    stage('Build') {
      steps {
        script {
          // Build image with registry/repo:tag so it can be pushed later
          app = docker.build("${registry}/${repo}:${tag}")
        }
      }
    }

    stage('Push') {
      steps {
        script {
          // Option A: Login to ECR with AWS CLI (recommended if aws-cli is available on agent)
          sh "aws ecr get-login-password --region us-east-1ß | docker login --username AWS --password-stdin ${registry}"
          app.push()

          // Option B: If you have configured a Jenkins credential / ECR credential helper,
          // uncomment and use docker.withRegistry instead of the aws-cli login above:
          // docker.withRegistry("https://${registry}", 'ecr:us-west-2:aws-credentials') {
          //   app.push()
          // }
        }
      }
    }
  }
}