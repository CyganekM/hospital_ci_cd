pipeline {
  environment {
    REGISTRY_DEPS = "jfrog.it-academy.by/public/hospital/deps"
    REGISTRY_APP = "jfrog.it-academy.by/public/hospital/app"
    registryCredential = 'jfrog_sa'
    DOCKERFILE_DEPS = 'Dockerfile_deps'
    DOCKERFILE_APP = 'Dockerfile_from_deps'

  }

  agent {
    label 'docker'
  }

  stages {
    stage('Clone repository') {
      steps {
        deleteDir()
        git(
          url: 'https://github.com/CyganekM/hospital.git',
          credentialsId: "token_github",
          branch: 'main',
          changelog: true,
          poll: true
        )
      }
    }

    stage('Check image exists in registry') {
      steps {
        script {
          withCredentials([usernamePassword(
            credentialsId: 'jfrog_sa',
            usernameVariable: 'JFROG_USER',
            passwordVariable: 'JFROG_PASSWORD'
          )]) {
            def fileExists = sh(script: '''
              # Проверяем HTTP статус код
              STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
                -u $JFROG_USER:$JFROG_PASSWORD \
                "https://jfrog.it-academy.by/artifactory/public/hospital/deps/latest/list.manifest.json")

              if [ "$STATUS" = "200" ]; then
                echo "EXIST"
              else
                echo "NO_EXIST"
              fi
            ''', returnStdout: true).trim()

            env.LATEST_EXISTS = fileExists
            echo "Manifest file exists: ${env.LATEST_EXISTS}"
          }
        }
      }
    }

    stage('Check for pom.xml changes') {
      when {
        expression {
          env.LATEST_EXISTS == "EXIST"
        }
      }
      steps {
        script {
          def changes = sh(script: '''
            if git diff --name-only HEAD~1 HEAD 2>/dev/null | grep -q "pom.xml"; then
              echo "CHANGED"
            else
              echo "UNCHANGED"
            fi
          ''', returnStdout: true).trim()

          env.POM_CHANGED = changes
          echo "pom.xml changed: ${env.POM_CHANGED}"
        }
      }
    }

    stage('Building image dependency') {
      when {
        expression {
          env.POM_CHANGED == 'CHANGED' || env.LATEST_EXISTS == "NO_EXIST"
        }
      }
      steps {
        script {
          echo "Building Docker image dependency..."
          echo "Reason: ${env.POM_CHANGED == 'CHANGED' ? 'pom.xml changed' : 'latest image not found in registry'}"

          dockerImage = docker.build(
            "${REGISTRY_DEPS}:${BUILD_NUMBER}",
            "-f ${DOCKERFILE_DEPS} ."
          )
          sh "docker tag ${REGISTRY_DEPS}:${BUILD_NUMBER} ${REGISTRY_DEPS}:latest"
        }
      }
    }

    stage('Push Image dependency') {
      when {
        expression {
          env.POM_CHANGED == 'CHANGED' || env.LATEST_EXISTS == "NO_EXIST"
        }
      }
      steps {
        script {
          docker.withRegistry('https://jfrog.it-academy.by/', registryCredential) {
            dockerImage.push()
            dockerImage.push('latest')
          }

          echo "✅ Image dependency pushed successfully:"
          echo "   ${REGISTRY_DEPS}:${BUILD_NUMBER}"
          echo "   ${REGISTRY_DEPS}:latest"
        }
      }
    }

        stage('Building image application') {
          steps {
            script {
              echo "Building Docker image application..."
              dockerImage = docker.build(
                "${REGISTRY_APP}:${BUILD_NUMBER}",
                "-f ${DOCKERFILE_APP} ."
              )
              sh "docker tag ${REGISTRY_APP}:${BUILD_NUMBER} ${REGISTRY_APP}:latest"
            }
          }
        }

         stage('Push Image application') {
           steps {
             script {
               docker.withRegistry('https://jfrog.it-academy.by/', registryCredential) {
                 dockerImage.push()
                 dockerImage.push('latest')
               }

               echo "✅ Image dependency pushed successfully:"
               echo "   ${REGISTRY_APP}:${BUILD_NUMBER}"
               echo "   ${REGISTRY_APP}:latest"
             }
           }
         }

stage('Update deployment manifest') {
  steps {
    script {
      echo "Updating k8s/hospital-deployment.yaml with new image tag..."
      sh """
        sed -i 's|image: jfrog.it-academy.by/public/hospital/app.*|image: ${REGISTRY_APP}:${BUILD_NUMBER}|' k8s/hospital-deployment.yaml
      """
      sh 'cat k8s/hospital-deployment.yaml | grep "image:"'
    }
  }
}

stage('Commit and push to Git') {
  steps {
    script {
      echo "Committing and pushing changes to Git repository..."

      // Используем GitHub credentials напрямую
      withCredentials([
        usernamePassword(
          credentialsId: 'token_github',
          usernameVariable: 'GIT_USERNAME',
          passwordVariable: 'GIT_PASSWORD'
        )
      ]) {
        sh """
          git config --global user.email "jenkins@example.com"
          git config --global user.name "Jenkins CI/CD"
          git add k8s/hospital-deployment.yaml
          git commit -m "CI/CD: Update image tag to ${BUILD_NUMBER} [Build #${BUILD_NUMBER}]"
          git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/CyganekM/hospital.git HEAD:main
        """
        }
      echo "✅ Changes pushed to Git repository"
    }
  }
}


    stage('Remove Unused docker image') {
      steps {
        sh '''
          # Безопасное удаление всех образов из нашего registry
          docker rmi $(docker images --filter=reference="jfrog.it-academy.by/*" -q) 2>/dev/null || true

          # Очищаем dangling images
          docker image prune -f
        '''
      }
    }
}