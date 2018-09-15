pipeline {
    agent any
    environment {
      ORG               = 'rawlingsj'
      APP_NAME          = 'acme-trading-frontend'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
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

            sh "npm install"
            sh "CI=true DISPLAY=:99 npm test"

            sh 'export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml'

            sh "jx step validate --min-jx-version 1.2.36"
            sh "jx step post build --image \$JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:\$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:$PREVIEW_VERSION"


          dir ('./charts/preview') {

             sh "make preview"
             sh "jx preview --app $APP_NAME --dir ../.."

          }
        }
      }
      stage('Build Release') {
        when {
          branch 'master'
        }
        steps {
          git 'https://github.com/rawlingsj/acme-trading-frontend.git'

          sh "git config --global credential.helper store"
          sh "jx step validate --min-jx-version 1.1.73"
          sh "jx step git credentials"
          // so we can retrieve the version in later steps
          sh "echo \$(jx-release-version) > VERSION"

          dir ('./charts/acme-trading-frontend') {
            sh "make tag"
          }

          sh "npm install"
          sh "CI=true DISPLAY=:99 npm test"

          sh 'export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml'
          sh "jx step validate --min-jx-version 1.2.36"
          sh "jx step post build --image \$JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:\$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:\$(cat VERSION)"

        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/acme-trading-frontend') {

            sh 'jx step changelog --version v\$(cat ../../VERSION)'

            sh 'make release'
            // promote through all 'Auto' promotion Environments
            sh 'jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION) --no-wait'
          }
        }
      }
    }
  }
