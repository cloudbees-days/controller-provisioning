def event = currentBuild.getBuildCauses()[0].event
library 'pipeline-library'
pipeline {
  agent none
  environment {
    OPS_PROVISION_SECRET = credentials('casc-workshop-controller-provision-secret')
    CONTROLLER_PROVISION_SECRET = event.secret.toString()
  }
  options { timeout(time: 10, unit: 'MINUTES') }
  triggers {
    eventTrigger jmespathQuery("controller.action=='provision'")
  }
  stages {
    stage('Provision Managed Controller') {
      agent {
        kubernetes {
          yaml libraryResource ('podtemplates/kubectl.yml')
        }
      }
      environment {
        ADMIN_CLI_TOKEN = credentials('admin-cli-token')
        WORKSHOP_ID="cloudbees-ci-casc-workshop"
        GITHUB_ORGANIZATION = event.github.organization.toString().replaceAll(" ", "-")
        GITHUB_REPOSITORY = event.github.repository.toString().toLowerCase()
        CONTROLLER_FOLDER = GITHUB_ORGANIZATION.toLowerCase()
        BUNDLE_ID = "${CONTROLLER_FOLDER}-${GITHUB_REPOSITORY}"
        AVAILABILITY_PATTERN = "${WORKSHOP_ID}/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}"
      }
      when {
        triggeredBy 'EventTriggerCause'
        environment name: 'CONTROLLER_PROVISION_SECRET', value: OPS_PROVISION_SECRET
      }
      steps {
        container("kubectl") {
          sh '''
            rm -rf ./${BUNDLE_ID} || true
            rm -rf ./checkout || true
            mkdir -p ${BUNDLE_ID}
            mkdir -p checkout
            git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git checkout
          '''
          dir('checkout/bundle') {
            sh '''
              cp --parents `find -name \\*.yaml*` ../../${BUNDLE_ID}//
            '''
          }
          sh '''
            ls -la ${BUNDLE_ID}
            kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_config/jcasc-bundles-store/ -c jenkins
          '''
        }
        sh '''
          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            http://cjoc/cjoc/load-casc-bundles/checkout

          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            http://cjoc/cjoc/casc-items/create-items?path=/$WORKSHOP_ID \
            --data-binary @./checkout/controller.yaml -H 'Content-Type:text/yaml'
        '''
      }
    }
  }
}
