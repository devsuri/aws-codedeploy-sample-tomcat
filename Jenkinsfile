#!groovy
pipeline {
    agent any
    /*parameters { 
	    choice(name: 'CHOICES', choices: ['one', 'two', 'three'], description: 'Deployment Portion') 
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
	stage("Release scope") {
            steps {
          step([$class: 'AWSCodeDeployPublisher', applicationName: 'CodeDeploy', awsAccessKey: '${env.AWS_ACCESS_KEY_ID}', awsSecretKey: '${env.AWS_SECRET_ACCESS_KEY_ID}', 
				  credentials: 'awsAccessKey', deploymentGroupAppspec: false, deploymentGroupName: 'codedeploygroup', 
				  deploymentMethod: 'CodeDeployDefault.AllAtOnce', includes: '**', proxyHost: '', 
				  proxyPort: 0, region: 'us-east-1', s3bucket: 'aws-code-deploy-test-jenkins', s3prefix: '', subdirectory: '', 
				  versionFileName: '', waitForCompletion: false])  
		    
		
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
    //slackUploadFile filePath: '/var/jenkins_home/workspace/pipeline-jenkinstest/arachni_report.html.zip', initialComment:  'HEY HEY'
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
