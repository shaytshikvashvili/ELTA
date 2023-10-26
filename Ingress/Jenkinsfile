pipeline {
  agent {
    label 'kubeagent'
  }
  stages {
    stage('Build and Deliver') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor --context `pwd` --dockerfile Dockerfile --destination shayts/elta-project:${BUILD_NUMBER}
          '''
        }
      }
    }
    stage('Deploy') {
      steps {
        container('k8s') {
          sh 'sed -i "s,IMAGE_NAME,shayts/elta-project:${BUILD_NUMBER}," netcore-app-deployment.yaml'
          sh "kubectl apply -f netcore-app-deployment.yaml -n prod"
        }
      }
    }
  }
}
