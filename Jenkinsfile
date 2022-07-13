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
        
        stage('Version Check') {
        steps {
            sh "npm -v"
            sh "mvn -v"
        }}

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
            }
          }
        }

        stage('Install dependencies') {
          steps {
              sh "npm install --silent --no-fund"
            }
        }

        stage('Static Code Analysis') {
          steps {
            sh "./node_modules/eslint/bin/eslint.js --format html . > eslint-out.html"

            publishHTML(target: [
              allowMissing: false,
              alwaysLinkToLastBuild: false,
              keepAll: false,
              reportDir: ".",
              reportFiles: 'eslint-out.html',
              reportName: 'ESLint Report'
            ]);

            sh "rm eslint-out.html"

            sh "npm run apigeelint > apigeelint-out.html"

            publishHTML(target: [
              allowMissing: false,
              alwaysLinkToLastBuild: false,
              keepAll: false,
              reportDir: ".",
              reportFiles: 'apigeelint-out.html',
              reportName: 'Apigeelint Report'
            ]);

            sh "rm apigeelint-out.html"
          }
        }

        stage('Deploy X/hybrid') {
          when {
            expression { env.API_VERSION ==  'google'}
          }
          steps {
           // Token precedence: env var; jenkins-scope sa; vm-scope sa; token;

            script {
              if (env.GCP_SA_AUTH == "jenkins-scope") {
                 withCredentials([file(credentialsId: 'jenkis-gcp', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                   env.SA_TOKEN=sh(script:'gcloud auth print-access-token', returnStdout: true).trim()
                 }
              } else if (env.GCP_SA_AUTH == "token") {
                 env.SA_TOKEN=env.APIGEE_TOKEN
              } else { // vm-scope
                 env.SA_TOKEN=sh(script:'gcloud auth application-default print-access-token', returnStdout: true).trim()
              }

             wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: env.SA_TOKEN, var: 'SA_TOKEN']]]) {
               sh """
                 mvn clean install \
                   -Pgoogleapi \
                   -Denv="${env.APIGEE_ENV}" \
                   -Dbearer="${env.SA_TOKEN}" \
                   -Dorg="${env.APIGEE_ORG}"
                """
              }
            }
            }
        }
    }
}

