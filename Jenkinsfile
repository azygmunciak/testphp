pipeline {
  agent any
  stages{
    stage('Info-DEV') {
      steps {    
        script {       
          openshift.withCluster() {
            openshift.withProject("adrian-env-dev") {
              echo "Using project ${openshift.project()} in cluster ${openshift.cluster()}"
            }
          }
        }
      }   
    }    
    stage('Clear-DEV') {
     steps {    
        script {  
         openshift.withCluster() {
            openshift.withProject("adrian-env-dev") {
              openshift.selector( 'all', [ app:'testphp' ] ).delete()
            }     
          }
        }    
      }  
    }
    stage('Build app in DEV environment') {
      steps {    
        script {       
          openshift.withCluster() {
            openshift.withProject("adrian-env-dev") {
              def created = openshift.newApp( 'php~https://github.com/azygmunciak/testphp.git' )
              echo "new-app created ${created.count()} objects named: ${created.names()}"
              created.describe()
              
              def bc = created.narrow('bc')
              def result = bc.logs('-f')
              echo "The logs operation require ${result.actions.size()} oc interactions"
              echo "Logs executed: ${result.actions[0].cmd}"
              def logsString = result.actions[0].out
              def logsErr = result.actions[0].err  
            }
          }
        }    
      }    
    }
    stage('Info-PROD') {
      steps {       
        script {       
          openshift.withCluster() {
            openshift.withProject("adrian-env-prod") {
              echo "Using project ${openshift.project()} in cluster ${openshift.cluster()}"
            }
          }
        }    
      }    
    }
    stage('Build image in PROD') {
      steps {       
        script {       
          openshift.withCluster() {
            openshift.withProject("adrian-env-prod") {
              openshift.selector("bc", "myphpapp").startBuild("--wait=true")
            }
          }
        }  
      }    
    }
    stage('Tag Image') {
      steps {      
        script {       
          openshift.withCluster() {
           openshift.withProject("adrian-env-prod") {
             openshift.tag("myphpapp:latest", "myphpapp:${BUILD_NUMBER}")
           }
          }
        }       
      }    
    }
    stage ('Approve to deploy') {
      when {
        branch 'master'
      }
      options {
        timeout(time: 12, unit: 'HOURS')
      }      
      steps {
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
      when {
        branch 'master'
      }
      steps {        
        script {       
          openshift.withCluster() {
            openshift.withProject("adrian-env-prod") {
              openshift.selector("dc", "myphpapp").rollout().latest();
            }
          }
        }
      }
      post {
        success {
          echo "APP Updated successfully"
        }
        failure {
          echo "APP Update failed!"
        }
      }
    }
  }
}

