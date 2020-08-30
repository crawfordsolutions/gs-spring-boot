#!groovy

/*
 * Common Jenkinsfile used by all Maven Git projects and all branches.
 *
 * @author jcrawford
 *
 */
		
node {
	// Limit build concurrency to 1 per branch to fix "Could not perform a local checkout" (workspace@2) errors
	properties([disableConcurrentBuilds()])
  
	try {
        notifyBuild('STARTED') 
	 
        // Use openjdk to avoid greedy Oracle license prompt
        env.JAVA_HOME="/usr/java/jdk-11.0.1"
        env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"
        sh 'java -version'
        
		stage ('Checkout') {
			// Checkout using the SCM object to pass credential token w/o a password. 	    
			// See:  https://stackoverflow.com/questions/38769976/is-it-possible-to-git-merge-push-using-jenkins-pipeline.
			sh 'git config --global credential.helper cache'
			sh 'git config --global push.default simple'
			checkout([
				$class: 'GitSCM',
				branches: scm.branches,
				extensions: scm.extensions + [[$class: 'CleanCheckout']],
				userRemoteConfigs: scm.userRemoteConfigs
				])
		}
	       
		stage ('Build') {
			def mvnHome = tool 'apache-maven-3.3.9'
			def pom = readMavenPom file: 'pom.xml'
			def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")       
			// To avoid an infinite build loop in Git and merge conflicts, we run a Maven release build 
			// which does NOT bump the SNAPAHOT version.  Instead, we perform a release build for EVERY build, 
			// replace "-SNAPSHOT" with the Jenkins build number and push the tag to Git.
			// See:  https://www.cloudbees.com/blog/new-way-do-continuous-delivery-maven-and-jenkins-pipeline
    		sh "git fetch --tags"
    		sh "${mvnHome}/bin/mvn -DreleaseVersion=${version} -DdevelopmentVersion=${pom.version} -DpushChanges=false -DlocalCheckout=true -DpreparationGoals=initialize release:prepare release:perform -B"
			sh "git push --verbose --force origin refs/tags/${pom.artifactId}-${version}:refs/tags/${pom.artifactId}-${version}"
		}

		stage ('Post') {
			def pom = readMavenPom file: 'pom.xml'
			def version = pom.version.replace("-SNAPSHOT", ".${currentBuild.number}")       
			if (fileExists("${WORKSPACE}/target/checkout/target/${pom.artifactId}-${version}.tar")) {
			    sh "mkdir -p /var/www/html/${pom.artifactId}"
    			sh "cp -p ${WORKSPACE}/target/checkout/target/${pom.artifactId}-${version}.tar /var/www/html/${pom.artifactId}/"
			}			
			archiveArtifacts artifacts: '**/target/*.jar,**/target/*.tar,**/target/*.pom', excludes: '**/target/*sources*, **/target/*info*', onlyIfSuccessful: true
		}         
		 
	} catch (e) {
    	// If there was an exception thrown, the build failed
    	currentBuild.result = "FAILED"
    throw e
  	} finally {
    // Success or failure, always send notifications
    notifyBuild(currentBuild.result)
  }
} // End node

    
def notifyBuild(String buildStatus = 'STARTED') {
// build status of null means successful
buildStatus =  buildStatus ?: 'SUCCESSFUL'

// Default values
def colorName = 'RED'
def colorCode = '#FF0000'
def subject = "${buildStatus}: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}"
def summary = "${subject} (${env.BUILD_URL})"
def details = """${env.JOB_NAME} - Build #${env.BUILD_NUMBER}:
Check console output at: ${env.BUILD_URL}"""

// Override default values based on build status
if (buildStatus == 'STARTED') {
	color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESSFUL') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
}

// Send notifications
slackSend (color: colorCode, message: summary)

emailext(
	subject: subject,
    body: details,
    recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']]
    )
}