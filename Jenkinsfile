pipeline {
    agent {
      label "jenkins-maven"
    }
    environment {
      ORG               = 'jx-registry-test'
      APP_NAME          = 'demo'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
      HELM_HOME         = '/home/jenkins/.helm'
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container('maven') {
            sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
            sh "mvn install"
            sh 'export VERSION=$PREVIEW_VERSION && skaffold run -f skaffold.yaml'


            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }

          dir ('./charts/preview') {
           container('maven') {
             sh "make preview"
             sh "jx preview --app $APP_NAME --dir ../.."
           }
          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          container('maven') {
            // ensure we're not on a detached head
            sh "git checkout master"
            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
          }
          dir ('./charts/demo') {
            container('maven') {
              sh "make tag"
            }
          }
          container('maven') {
            sh 'mvn clean deploy'
            sh 'gcloud auth activate-service-account rafaelremondes@jx-registry-test.iam.gserviceaccount.com --key-file=/home/jenkins/.auth/JX-Registry-Test-84e5f80822db.json'


            sh 'export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml'
            sh 'helm plugin install https://github.com/hypnoglow/helm-s3.git'

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/demo') {
            container('maven') {
              sh 'jx step changelog --version v\$(cat ../../VERSION)'
              sh 'helm plugin install https://github.com/hypnoglow/helm-s3.git'

              // release the helm chart
              sh 'jx step helm release'

              // promote through all 'Auto' promotion Environments
              sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)'
            }
          }
        }
      }
    }
    post {
        always {
            cleanWs()
        }
        failure {
            input """Pipeline failed. 
We will keep the build pod around to help you diagnose any failures. 

Select Proceed or Abort to terminate the build pod"""
        }
    }
  }
