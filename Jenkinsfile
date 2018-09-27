pipeline {
    agent {
      label "jenkins-maven"
    }
    environment {
      ORG               = 'sarvanid'
      APP_NAME          = 'nexus'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
      MAVEN_HOME        = '$M2_HOME'
      JAVA_HOME         = "/usr/lib/jvm/java"
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
          configArtifactoryStage()
          container('maven') {
            runMaven("versions:set -DnewVersion=" + "$PREVIEW_VERSION")
            runMaven("install")
            sh 'export VERSION=$PREVIEW_VERSION && skaffold run -f skaffold.yaml'

            sh "jx step validate --min-jx-version 1.2.36"
            sh "jx step post build --image \$JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:\$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:$PREVIEW_VERSION"
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
          configArtifactoryStage()
          container('maven') {
            // ensure we're not on a detached head
            sh "git checkout master"
            sh "git config --global credential.helper store"
            sh "jx step validate --min-jx-version 1.1.73"
            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            runMaven("versions:set -DnewVersion=\$(jx-release-version)")
          }
          dir ('./charts/nexus') {
           container('maven') {
              sh "make tag"
            }
          }
          container('maven') {
            deploy()

            sh 'export VERSION=`cat VERSION` && skaffold run -f skaffold.yaml'

            sh "jx step validate --min-jx-version 1.2.36"
            sh "jx step post build --image \$JENKINS_X_DOCKER_REGISTRY_SERVICE_HOST:\$JENKINS_X_DOCKER_REGISTRY_SERVICE_PORT/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
      stage('Promote to Environments') {
        when {
          branch 'master'
        }
        steps {
          dir ('./charts/nexus') {
            container('maven') {
              sh 'jx step changelog --version v\$(cat ../../VERSION)'

              // release the helm chart
              sh 'make release'

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

//def buildInfo
  //  }
//}

void runMaven(goals) {
    script {
      
        sh "mvn " + goals
      }
    }


void deploy() {
    script {
      
        runMaven("clean deploy")
      }
    }

