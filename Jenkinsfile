pipeline {
  agent any
  environment {
    DEV_ENVIRONMENT = 'adrian-env-dev'
    PROD_ENVIRONMENT = 'adrian-env-prod'
    DEV_APP_NAME = 'testphp'
    PROD_APP_NAME = 'myphpapp'
  }
  stages{
    stage('Info-DEV') {
      steps {    
        script {       
          openshift.withCluster() {
            openshift.withProject("$DEV_ENVIRONMENT") {
              echo "Using project ${openshift.project()} in cluster ${openshift.cluster()}"
              slackSend channel: 'my-test-channel', message: "Starting new build of $DEV_APP_NAME using project ${openshift.project()} in cluster ${openshift.cluster()}"
            }
          }
        }
      }   
    }    
    stage('Clear-DEV') {
     steps {    
        script {  
         openshift.withCluster() {
            openshift.withProject("$DEV_ENVIRONMENT") {
              openshift.selector( 'all', [ app:'$DEV_APP_NAME' ] ).delete()
            }     
          }
        }    
      }  
    }
    stage('Build app in DEV environment') {
      steps {    
        script {       
          openshift.withCluster() {
            openshift.withProject("$DEV_ENVIRONMENT") {
              def created = openshift.newApp( 'php~https://github.com/azygmunciak/testphp.git' )
              echo "new-app created ${created.count()} objects named: ${created.names()}"
              created.describe()
              
              def bc = created.narrow('bc')
              def result = bc.logs('-f')
              echo "The logs operation require ${result.actions.size()} oc interactions"
              echo "Logs executed: ${result.actions[0].cmd}"
              def logsString = result.actions[0].out
              def logsErr = result.actions[0].err  

              openshift.expose("svc/testphp","--hostname=bookinfo-web-bookinfo.apps.test.oc.io")
            }
          }
        }    
      }    
    }
    stage('Test-DEV') {
      options {
        timeout(time: 3, unit: 'MINUTES')
      }        
      steps {    
        sh(script: '''
          TEST=0
          while [[ $TEST -ne 1 ]]; do
            TEST=$(curl http://bookinfo-web-bookinfo.apps.test.oc.io/health.html 2> /dev/null|grep alive|wc -l)
            sleep 2
          done
          ''')    
      }  
      post {
        success {
          echo "APP tested successfully"
        }
        failure {
          echo "APP test failed!"
        }
      }        
    }    
    stage('Info-PROD') {
      steps {       
        script {    
          openshift.withCluster() {
            openshift.withProject("$PROD_ENVIRONMENT") {
              echo "Using project ${openshift.project()} in cluster ${openshift.cluster()}"
              slackSend channel: 'my-test-channel', message: "Starting new build of $PROD_APP_NAME using project ${openshift.project()} in cluster ${openshift.cluster()}"
            }
          }
        }    
      }    
    }
    stage('Build image in PROD') {
      steps {       
        script {       
          openshift.withCluster() {
            openshift.withProject("$PROD_ENVIRONMENT") {
              openshift.selector("bc", "$PROD_APP_NAME").startBuild("--wait=true")
            }
          }
        }  
      }    
    }
    stage('Tag Image') {
      steps {      
        script {       
          openshift.withCluster() {
           openshift.withProject("$PROD_ENVIRONMENT") {
             openshift.tag("$PROD_APP_NAME:latest", "$PROD_APP_NAME:${BUILD_NUMBER}")
           }
          }
        }       
      }    
    }
    stage ('Approve to deploy') {
      //when {
      //  branch 'master'
      //}
      options {
        timeout(time: 4, unit: 'HOURS')
      }      
      steps {
        openshift.withCluster() {
          openshift.withProject("$PROD_ENVIRONMENT") {
            slackSend channel: 'my-test-channel', message: "Waiting for approval: $PROD_APP_NAME using project ${openshift.project()} in cluster ${openshift.cluster()}"
          }
        }
        input message: "Deploy changes to PROD?"
      }
      post {
        success {
          echo "Deployment approved."
        }
        failure {
          echo "Deployment rejected!"
        }    
      }  
    }
    stage('Deploy APP in PROD') {
      //when {
      //  branch 'master'
      //}
      steps {        
        script {       
          openshift.withCluster() {
            openshift.withProject("$PROD_ENVIRONMENT") {
              openshift.selector("dc", "$PROD_APP_NAME").rollout().latest();
            }
          }
        }
      }
    }
    stage('Test-PROD') {
      options {
        timeout(time: 5, unit: 'MINUTES')
      }        
      steps {    
        sh(script: '''
          TEST=0
          while [[ $TEST -ne 1 ]]; do
            TEST=$(curl http://robot-shop-web-robot-shop.apps.test.oc.io/health.html 2> /dev/null|grep alive|wc -l)
            sleep 2
          done
          ''')    
      }  
      post {
        success {
          script { 
            echo "APP updated successfully"
            openshift.withCluster() {
              openshift.withProject("$PROD_ENVIRONMENT") {  
                slackSend channel: 'my-test-channel', message: "Application $PROD_APP_NAME in ${openshift.project()} project has been updated successfully!"
              }
            }
          }
        }
        failure {
          script {           
            echo "APP update failed"
            openshift.withCluster() {              
              openshift.withProject("$PROD_ENVIRONMENT") {            
                slackSend channel: 'my-test-channel', message: "DEPLOYMENT FAILED!!! Application $PROD_APP_NAME in ${openshift.project()} project has been updated unsuccessfully!!!"
              }
            }
          }
        }
      }      
    }       
  }
}

