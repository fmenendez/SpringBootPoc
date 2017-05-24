#!/usr/bin/env groovy

import groovy.json.JsonOutput

/*
 * Retrieve current SCM information from local checkout
 */
def getSCMInformation() {
    def gitRemoteUrl = ''
    try{
        gitRemoteUrl = sh(returnStdout: true, script: 'git config --get remote.origin.url').trim()
    }catch (e){}
    
    def gitCommitSha = '';
    try{
        gitCommitSha = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    }catch(e){}
    
    def gitBranchName = '';
    try{
        gitBranchName = sh(returnStdout: true, script: 'git name-rev --always --name-only HEAD').trim().replace('remotes/origin/', '')
    }catch(e){}

    return [
        url: gitRemoteUrl,
        branch: gitBranchName,
        commit: gitCommitSha
    ]
}

/*
 * Notify the Circuit services about the status of a build based from a
 * git repository.
 */
def notifyCircuit(buildStatus, buildPhase="FINALIZED", endpoint="http://48e78108.ngrok.io/notifications/jenkins/") {
	try{
    def payload = JsonOutput.toJson([
        name: env.JOB_NAME,
        duration: currentBuild.duration,
				url: env.JOB_URL.substring(env.JENKINS_URL.size()),
        build      : [
            number: env.BUILD_NUMBER,
            phase: buildPhase,
            status: buildStatus,
            full_url: env.BUILD_URL,
            scm: getSCMInformation(),
	    			url: env.BUILD_URL.substring(env.JENKINS_URL.size())
        ]
    ])
		
   	sh("curl --silent -XPOST -H 'Content-Type: application/json' -d '${payload}' ${endpoint}")
	}catch(e){}
}

node ('master'){
	try {
		notifyCircuit("", "STARTED")

		stage('Download code') {
			checkout scm
    }
    
    stage('Build, Test and Deploy') {
            sh("docker build -t io.thinkd.poc.springbootpoc -f docker/BuildDockerfile .")
            sh("docker run -v /var/maven/.m2:/var/maven/.m2 -v /var/run/docker.sock:/var/run/docker.sock -e MAVEN_CONFIG=/var/maven/.m2 --network proxy io.thinkd.poc.springbootpoc")
    }
	
    notifyCircuit("", "BUILD_COMPLETE")
    input id: 'sign', message: 'Sign?'

	/*stage('Kubernetes Deploy') {
	    def workspace = pwd()
        if (isUnix()) {
			sh "kubectl replace -f ${workspace}/Kubernetes/kubernetes.yaml --force"
		}else{
			bat "kubectl replace -f ${workspace}/Kubernetes/kubernetes.yaml --force"
		}
    }*/

		notifyCircuit("SUCCESS")
  }catch(hudson.AbortException e){
		notifyCircuit("ABORTED")
		throw e
  }catch(java.lang.InterruptedException e){
		notifyCircuit("ABORTED")
		throw e
  }catch(Exception e){	   
 	  def m = (e.message =~ /(?i)script returned exit code (\d+)/)
		if (m) {
		    def exitcode = m.group(1).toInteger()
		    m = null
		    if (exitcode >= 127) {
			    notifyCircuit("ABORTED")
			    throw e;    //  killed because of abort, letting through
		    }
    }
  	notifyCircuit("FAILURE")
	  throw e
  }
}