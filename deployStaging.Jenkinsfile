@Library('ace@master') _

def tagMatchRules = [[
  "meTypes": [
    "SERVICE"
  ],
  tags: [
    ["context": "CONTEXTLESS", "key": "app", "value": "simplenodeservice"],
    ["context": "CONTEXTLESS", "key": "environment", "value": "staging"]
  ]
]]

pipeline {
  parameters {
    string(name: 'APP_NAME', defaultValue: 'simplenodeservice', description: 'The name of the service to deploy.', trim: true)
    string(name: 'TAG_STAGING', defaultValue: '', description: 'The image of the service to deploy.', trim: true)
    string(name: 'BUILD', defaultValue: '', description: 'The version of the service to deploy.', trim: true)
  }
  agent {
    label 'kubegit'
  }
  stages {
    stage('Update Deployment and Service specification') {
      steps {
        script {
          env.DT_CUSTOM_PROP = readFile "manifests/staging/dt_meta"
          env.DT_CUSTOM_PROP = env.DT_CUSTOM_PROP + " " + generateDynamicMetaData()
        }
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'git-creds-ace', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
            sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
            sh "echo ${env.DT_CUSTOM_PROP}"
            sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/maas-hot"
            sh "cd maas-hot/ && sed 's#value: \"DT_CUSTOM_PROP_PLACEHOLDER\".*#value: \"${env.DT_CUSTOM_PROP}\"#' manifests/${env.APP_NAME}.yml > manifests/staging/${env.APP_NAME}.yml"
            sh "cd maas-hot/ && sed -i 's#image: .*#image: ${env.TAG_STAGING}#' manifests/staging/${env.APP_NAME}.yml"
            sh "cd maas-hot/ && git add manifests/staging/${env.APP_NAME}.yml && git commit -m 'Update ${env.APP_NAME} version ${env.BUILD}'"
            sh "cd maas-hot/ && git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${env.GITHUB_ORGANIZATION}/maas-hot"
            sh "rm -rf maas-hot"
          }
        }
      }
    }
    stage('Deploy to staging namespace') {
      steps {
        checkout scm
        container('kubectl') {
          sh "kubectl -n staging apply -f manifests/staging/${env.APP_NAME}.yml"
        }
      }
    }
    stage('DT send Deployment event') {
      steps {
        script {
          def status = dt_pushDynatraceDeploymentEvent (
            tagRule : tagMatchRules,
            deploymentVersion: "Jenkins - ${env.BUILD}",
            customProperties : [
              "Jenkins Build Number": env.BUILD_ID,
              "Git commit": env.GIT_COMMIT
            ]
          )
        }
      }
    }
    stage('DT send Info event') {
      steps {
        script {
          def status1 = dt_pushDynatraceInfoEvent (
            tagRule : tagMatchRules,
            deploymentVersion: "Jenkins - ${env.BUILD}",
            description: "The Coffee Machine Broke.",
            title: "No Coffee.",
            customProperties : [
              "Jenkins Build Number": env.BUILD_ID,
              "Git commit": env.GIT_COMMIT
            ]
          )
        }
      }
    }
    stage('DT send Config event') {
      steps {
        script {
          def status2 = dt_pushDynatraceConfigurationEvent (
            tagRule : tagMatchRules,
            deploymentVersion: "Jenkins - ${env.BUILD}",
            description: "Changed Coffee Filter",
            configuration: "Coffee Filter 2.0",
            customProperties : [
              "Jenkins Build Number": env.BUILD_ID,
              "Git commit": env.GIT_COMMIT
            ]
          )
        }
      }
    }
    // stage('DT send Info event') {
    //   steps {
    //       script {
    //         def info = dt_pushDynatraceInfoEvent (
    //           tagRule : tagMatchRules,
    //           customProperties : [
    //             [key: 'Jenkins Build Number', value: "${env.BUILD_ID}"],
    //             [key: 'Git commit', value: "${env.GIT_COMMIT}"]
    //           ],
    //           description: 'Info from Jenkins'
    //         )
    //       }
    //   }
    // }
    // stage('DT send Config event') {
    //   steps {
    //     script {
    //       def config = dt_pushDynatraceConfigurationEvent (
    //         tagRule : tagMatchRules,
    //         description: 'Config from Jenkins',
    //         configuration: 'Changed Things...',
    //         customProperties : [
    //           [key: 'Jenkins Build Number', value: "${env.BUILD_ID}"],
    //           [key: 'Git commit', value: "${env.GIT_COMMIT}"]
    //         ]
    //       )
    //     }
    //   }
    // }

    stage('Run tests') {
      steps {
        build job: "3. Test",
        parameters: [
          string(name: 'APP_NAME', value: "${env.APP_NAME}")
        ]
      }
    }
  }
}

  def generateDynamicMetaData(){
    String returnValue = "";
    returnValue += "SCM=${env.GIT_URL} "
    returnValue += "Branch=${env.GIT_BRANCH} "
    returnValue += "Build=${env.BUILD} "
    returnValue += "Image=${env.TAG_STAGING} "
    return returnValue;
  }
