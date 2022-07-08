pipeline {

    agent any

    parameters {
            string(name: 'API_VERSION_P', defaultValue: 'google')
            string(name: 'APIGEE_ORG_P', defaultValue: 'my-org')
            string(name: 'APIGEE_TEST_ENV_P', defaultValue: 'test1')
            choice(name: 'GCP_SA_AUTH_P', choices: [ "vm-scope", "jenkins-scope", "token" ], description: 'GCP SA/Token Scope'  )
    }

    environment {
        GOOGLE_APPLICATION_CREDENTIALS= credentials('jenkis-gcp')
        // Mutliple options for setting the Apigee deployment target config:
        // 1. As a jenkins global property at ${JENKINS_URL}/configure if you don't have access to edit this file
        // 2. As a branch specific environment variable in the first pipeline stage
        // 3. As a environment variable for all branches (see below)

        API_VERSION = 'google'
        APIGEE_ORG = 'ehumar-apigee-terra'
        APIGEE_ENV = 'test'
        TEST_HOST= 'test.apigee.net'
        APIGEE_DEPLOYMENT_SUFFIX='jenkis'
        AUTHOR_EMAIL = '@google.com'
    }
    
    stages {
        stage("Env Variables") {
            steps {
                sh "printenv"
            }
        }
        stage('Set Apigee Env and Proxy Suffix') {
          steps {	
            script{
              // Main branch for Apigee test environment
              if (env.GIT_BRANCH == "main") {
                  env.APIGEE_DEPLOYMENT_SUFFIX = ""
                  env.APIGEE_ENV = env.APIGEE_TEST_ENV
                  // env.APIGEE_ORG = 'apigee-org-name'
              // Prod branch for Apigee prod environment
              } else if (env.GIT_BRANCH == "prod") {
                  env.APIGEE_DEPLOYMENT_SUFFIX = ""
                  env.APIGEE_ENV = env.APIGEE_PROD_ENV
                  // env.APIGEE_ORG = 'apigee-org-name'
              // All other branches are deployed as separate proxies with suffix in the test environment
              } else {
                  env.APIGEE_DEPLOYMENT_SUFFIX = env.GIT_BRANCH ? "-" + env.GIT_BRANCH.replaceAll("\\W", "-") : "-jenkins"
                  env.APIGEE_ENV = env.APIGEE_TEST_ENV
                  // env.APIGEE_ORG = 'apigee-org-name'
              }
              println "---------- Branch-Dependent Build Config ----------"
              println "Apigee Git: " + env.GIT_BRANCH
              println "Apigee Org: " + env.APIGEE_ORG
              println "Apigee Env: " + env.APIGEE_ENV
              println "Proxy Deployment Suffix: " + env.APIGEE_DEPLOYMENT_SUFFIX
            }
          }
        }


        
        stage('Deploy X/hybrid') {
          when {
            expression { env.API_VERSION ==  'google'}
          }
          steps {
            sh '''
              #!/bin/bash
              APIGEE_SA_TOKEN="\${APIGEE_TOKEN:-\$(gcloud auth application-default print-access-token)}"
              mvn clean install \
                -Pgoogleapi \
                -Denv=${APIGEE_ENV} -Dorg=${APIGEE_ORG} -Dtoken=${APIGEE_SA_TOKEN} -Ddeployment.suffix=${APIGEE_DEPLOYMENT_SUFFIX} 
            '''
          }
        }
 
    }
}

