pipeline {
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Docker image tag')
  }

  agent any

  stages {
    stage('Clone repository') {
      steps {
        deleteDir()
        git(
          url: 'https://github.com/CyganekM/hospital_ci_cd.git',
          credentialsId: "token_github",
          branch: 'main',
          changelog: true,
          poll: true
        )
      }
    }

    stage('Push changes') {
      steps {
        script {
          withCredentials([usernamePassword(
            credentialsId: 'token_github',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_PASS'
          )]) {
            sh """
              sed -i 's|image:.*jfrog.it-academy.by/public/hospital/app.*|image: ${params.IMAGE_TAG}|' k8s/hospital-deployment.yaml
              git add k8s/hospital-deployment.yaml
              git commit -m "Update image to ${params.IMAGE_TAG}"
              git push https://${GIT_USER}:${GIT_PASS}@github.com/CyganekM/hospital_ci_cd.git main
              echo "✅ Successfully updated deployment!"
            """
          }
        }
      }
    }
  }

  post {
    always {
      echo "Deployment update completed for: ${params.IMAGE_TAG}"
    }
  }
}