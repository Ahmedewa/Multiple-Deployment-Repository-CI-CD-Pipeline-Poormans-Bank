pipeline {
  agent any

  environment {
    DOCKER_IMAGE = 'poormansbank'
    JIRA_CLUSTER = 'poormansbank'
    JIRA_SERVICE = 'poormansbank'
    AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    AZURE_CREDENTIALS = credentials('azure-credentials')
    GCP_CREDENTIALS = credentials('gcp-credentials')
  }

  stages {
    stage('Build') {
      steps {
        sh 'docker build -t ${DOCKER_IMAGE} .'
      }
    }
    stage('Test') {
      steps {
        sh 'docker run -p 8080:8080 ${DOCKER_IMAGE} npm test'
      }
    }
    stage('Deploy to AWS') {
      steps {
        withAWS(credentials: 'aws-credentials') {
          sh 'aws ecs update-service --cluster ${JIRA_CLUSTER} --service ${JIRA_SERVICE} --force-new-deployment'
        }
      }
    }
    stage('Deploy to Azure') {
      steps {
        withCredentials([azureServicePrincipal('azure-credentials')]) {
          sh 'az webapp deployment source config-zip --resource-group myResourceGroup --name myAppName --src myapp.zip'
        }
      }
    }
    stage('Deploy to GCP') {
      steps {
        withCredentials([file(credentialsId: 'gcp-credentials', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
          sh 'gcloud app deploy'
        }
      }
    }
  }
}
