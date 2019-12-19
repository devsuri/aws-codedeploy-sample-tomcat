#!groovy
pipeline {
    agent any
    /*parameters { 
	   // choice(name: 'CHOICES', choices: ['one', 'two', 'three'], description: 'Deployment Portion') 
    }*/
    options {
    buildDiscarder(logRotator(numToKeepStr: '3'))
	}
    tools {
    maven 'M2_HOME'
    }
    
    stages {
        stage('Build') {
            steps {
		    // Run the maven build
		    echo 'Clean Build'
		sh 'mvn -B -DskipTests clean package'
            }
        }
	 
	  stage('Test') {
            steps {
		    // Run integration test
                echo 'Testing'
                sh 'mvn test'
            }
         }
	    
	    stage('Sonar-Static-Scanning'){
	        steps {
		        // //Run the sonar scan
                echo 'Sonar Scanner'
		        // sh "mvn sonar:sonar -Dsonar.host.url=http://198.198.10.46:9000 -Dsonar.login=XXXXXXXXX"
		        withSonarQubeEnv('SonarQube') {
                    sh " mvn verify sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
	            }
            }
	    }
		
        // waiting for sonar results based into the configured web hook in Sonar server which push the status back to jenkins
	    stage('Sonar scan result check') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    retry(3) {
                        script {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }

        }
	    stage("Release scope") {
            steps {
                script {
                    
                    env.RELEASE_SCOPE = input message: 'User input required', ok: 'Release!',
                    parameters: [choice(name: 'RELEASE_SCOPE', 
                    choices: ['Shell','AWSCodeDeploy', 'AzureAppService'], 
                    description: 'What is the release scope?')]
               
                //echo "Release scope selected: ${env.RELEASE_SCOPE}"
		    
			    if (env.RELEASE_SCOPE == "Shell"){
			        echo "execute Shell"
			        sshagent(['dev-server']) {
		                sh "/var/jenkins_home/workspace/backupscript/backup.sh"
                                sh "rsync -ivhr $WORKSPACE/target/SampleMavenTomcatApp.war -e 'ssh -o StrictHostKeyChecking=no' '${env.codedeployserver}':'/tmp/shell/'"
                            }
                   }
		    
		    
			    if (env.RELEASE_SCOPE == "AWSCodeDeploy") {
			        //provisioning Terraform for CodeDeploy
                                echo "Provisioning terraform"
			        //export AWS_ACCESS_KEY_ID="${env.AWS_ACCESS_KEY_ID}"
			        //export AWS_SECRET_ACCESS_KEY="${env.AWS_SECRET_ACCESS_KEY_ID}"
			        sh "cp /var/jenkins_home/workspace/terraform/* $WORKSPACE/"
			        sh "terraform init"
			        sh "terraform apply -auto-approve"
			        echo "execute AWSCodeDeploy"
                                //Publishing Artifacts through CodeDeploy
                                step([$class: 'AWSCodeDeployPublisher', applicationName: 'CodeDeploy', 
                                awsAccessKey: '${env.AWS_ACCESS_KEY_ID}', awsSecretKey: '${env.AWS_SECRET_ACCESS_KEY_ID}', 
				deploymentGroupAppspec: false, deploymentGroupName: 'codedeploygroup', 
				deploymentMethod: 'CodeDeployDefault.AllAtOnce', includes: '**', proxyHost: '', 
		                proxyPort: 0, region: 'us-east-1', s3bucket: 'aws-code-deploy-test-jenkins'])  
		            }
			
			    if (env.RELEASE_SCOPE == "AzureAppService") {
			        echo "execute AzureAppService"
			        azureWebAppPublish appName: 'codedeploy-appservice', azureCredentialsId: 'yourAzureServicePrincipalName',  
				filePath: 'SampleMavenTomcatApp.war', publishType: 'file', resourceGroup: 'codedeploy', slotName: '', 
				sourceDirectory: 'target', targetDirectory: '/'
                            }
		     }
               }
          }
   }

    post {
        success {
            notifyBuild()
        }
	    
        
	    failure {
            notifyBuild('ERROR')
	    }
    }
}

// Slack notification with status and code changes from git
def notifyBuild(String buildStatus = 'SUCCESSFUL') {
    //buildStatus = buildStatus
    buildStatus =  buildStatus ?: 'SUCCESSFUL'

    def colorName = 'RED'
    def colorCode = '#FF0000'
    def subject = "${buildStatus}: Job Name - '${env.JOB_NAME}', Build No. - '${env.BUILD_NUMBER}', Git Commit - '${env.GIT_COMMIT}'"
    def changeSet = getChangeSet()
    def message = "${subject} \n ${changeSet}"

    if (buildStatus == 'SUCCESSFUL') {
        color = 'GREEN'
        colorCode = '#00FF00'
    } 
    else {
        color = 'RED'
        colorCode = '#FF0000'
    }

    slackSend(color: colorCode, message: message)
    
}

// Fetching change set from Git
@NonCPS
def getChangeSet() {
    return currentBuild.changeSets.collect { cs ->
    cs.collect { entry ->
            "* ${entry.author.fullName}: ${entry.msg}"
        }.join("\n")
    }.join("\n")
}
