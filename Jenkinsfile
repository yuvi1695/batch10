import jenkins.model.*
      
try{      
      node{
            def mvnHome = tool name: 'MAVEN3'
            def dockerHome = tool name: 'DOCKER'  
            def version = 1.0


              stage ('Code Checkout') {
                  echo "Checking out code from GitHub..."
                  checkout scm
              }

              stage ('Build and Package') {
                  echo "Building the Application..."
                  sh "mvn clean package"
              }

             stage ('Sonar Scan') {
                   echo "Running Static Code Analysis..."
                   withSonarQubeEnv('SONARQUBE') {
                         sh "mvn sonar:sonar"
                   }
              }

            stage ('Security Scan') {
                  echo "Running Security Scan..."
                  def ws = pwd()
                  def artifact = findFiles(glob: '**/target/*.jar')
                  println ("Artifact to Scan :" + (artifact[0].path))
                  appscan application: 'b9e4f0c1-018b-497f-801e-7d9330303377', credentials: 'app-scan', name: 'Security-test', scanner: static_analyzer(hasOptions: false, target: "${ws}/${artifact[0].path}"), type: 'Static Analyzer', wait: true
            }
            
            stage ('Publish HTML Reports') {
                  echo " Publishing HTML Reports..."
                  publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: '', reportFiles: 'index.html', reportName: 'HTML Report', reportTitles: ''])
            }

            withCredentials([usernamePassword(credentialsId: 'dockerCreds', passwordVariable: 'dockerPwd', usernameVariable: 'dockerUser')]) {
                  stage ('Build Docker Image') {
                        echo "Building Docker image for batch10 ..."
                        sh "docker build -t ${dockerUser}/batch10:${version} ."
                    }

                  stage("Push Docker Image to Docker Hub"){
                        echo "Pushing image..."
                        sh "docker login -u ${dockerUser} -p ${dockerPwd}"
                        sh "docker push ${dockerUser}/batch10:${version}"       
                    }

                    stage('Deploy Application'){
                          ansiColor('xterm') {
                              echo "Configuring the Instance and Deploying Application..."
                              ansiblePlaybook([
                                    credentialsId : 'ansibleCreds',
                                    disableHostKeyChecking : true,
                                    inventory     : '/etc/ansible/hosts',
                                    playbook      : 'deployment-playbook.yml',
                                    installation  : 'ANSIBLE',
                                    colorized     : true,
                                    extraVars     : [
                                          "user": "${dockerUser}",
                                          "version": "${version}"
                                    ]
                              ])
                          }
                    }
                  
                  stage ('Sending Notification') {
                        
                        echo "Sending Success Notification"
                        sendNotification("SUCCESS")
                        
                  }
            }
      }
}
catch (Exception e) {
      currentBuild.result = "FAILED"
      sendNotification("FAILED")
}
      
def sendNotification(buildStatus) {
                  
      def body = ""
                  
      if (buildStatus == "FAILED") {
            body = """<p>${buildStatus}: Application: ${env.JOB_NAME} with build [${env.BUILD_NUMBER}]:</p>
                  <p>Build has failed for the mentioned job. Please check the attached log for mre details.
                  <p></p>
                  <p></p>
                  <p>Thanks</p>
                  <p>DevOps Team</p>"""
      }
      else if (buildStatus == "SUCCESS") {
            body = """<p>${buildStatus}: Application: ${env.JOB_NAME} with build [${env.BUILD_NUMBER}]:</p>
                  <p>Build has been successfully completed for the mentioned job. Please check the attached log for more details.
                  <p></p>
                  <p></p>
                  <p>Thanks</p>
                  <p>DevOps Team</p>"""
      }
      
            
      emailext attachLog: true,attachmentsPattern: 'generatedFile.txt',body: body,subject: "Build for ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
            mimeType: 'text/html',to: 'yuvrajyuvi24@gmail.com'
}
