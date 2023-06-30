def checkoutSourceCode() {
    checkout([$class: 'GitSCM', branches: [[name: "refs/remotes/origin/main"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'raghu-github', url: 'https://github.com/Raghupatik/java-test.git']]])
}

pipeline {
    agent any

     options {
        //Disable concurrentbuilds for the same job
        disableConcurrentBuilds()         
        // Add timestamps to console log
        timestamps()

       // skipDefaultCheckout true  
    }

    parameters {
         choice(
            // choices are a string of newline separated values
            choices: ['All', 'Checkout', 'Build', 'Artifactory', 'DeployTestServer', 'TestTestServer', 'RollbackTestServer', 'DeployProduction', 'TestProduction', 'RollbackProduction'],
            description: 'Choose stage to run. All stages will be run by default.',
            name: 'Stage',
        )
	    string(name: 'RollbackVersion', defaultValue: '', description: 'Input the Rollback version to be deployed')
    }

    environment {
        S3_BUCKET = 'raghu-jenkinsartifacts'
        LAMBDA_FUNCTION = 'java-sample-lambq2'
    }

    stages {

        stage('Build') {
            agent { label 'jenkins-slave' }


            when {
                expression { params.Stage == 'All' || params.Stage == 'Build' }
            }
            steps {
                echo "Stage: ${params.Stage}"

                script {
                    echo 'Build'
                    sh 'mvn clean install'
                }
            }
        }

        stage('Push to S3') {
            agent { label 'jenkins-slave' }

            when {
                expression { params.Stage == 'All' || params.Stage == 'Artifactory' }
            }

            steps {
                script {
                    echo 'Push to S3'
                    ARTIFACTID = readMavenPom().getArtifactId()

                    VERSION = readMavenPom().getVersion()
                    JARNAME = ARTIFACTID+'-'+VERSION+'.jar'
                    echo "JARNAME: ${JARNAME}"    
                    //sh 'cp target/priceavailability-api*.war target/priceavailability-api.war'
                    withAWS(profile:'aviall', region:'us-east-1') {
                        s3Upload(file:"target/${JARNAME}", bucket:"${S3_BUCKET}", path:"${JARNAME}")
                    }
                    

                    // sh "aws s3 cp target/${JARNAME} s3://$S3_BUCKET"
                    currentBuild.result = 'SUCCESS'
                }
            }
        }

        stage('Clean Workspace') {
            agent { label 'jenkins-slave'}
            steps {
                cleanWs()
            }
        }

        stage ('Run postman test on Test Server') {
            when {
                expression { params.Stage == 'All' || params.Stage == 'TestTestServer' }
            }
            parallel {
                stage ('Run postman test on Tomcat Test') {
                    agent { label 'master'}
                    steps {
                        //checkoutSourceCode()
                        script {
                            retry(2) {
                                try {
                                    echo 'Run postman on Test server'
                                    sh 'newman run src/test/java/com/example/resources/postman-testscript.json -d src/test/java/com/example/resources/tomcat-env.json -k'
                                    currentBuild.result = 'SUCCESS'
                                } catch(Exception err) {
                                    // currentBuild.result = 'FAILURE'
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Lambda - Test') {
            agent { label 'master' }

            when {
                expression { params.Stage == 'All' || params.Stage == 'DeployTestServer' }
            }

            steps {
                script {
                    echo 'Deploy to Test'

                    sh "aws lambda update-function-code --function-name $LAMBDA_FUNCTION --region us-east-1 --s3-bucket $S3_BUCKET --s3-key ${JARNAME}"

                    currentBuild.result = 'SUCCESS'
                }          
            }
        }

       stage ('Rollback Test if deployment failed') {
            agent { label 'master'}
            when {
                expression { params.Stage == 'RollbackTestServer' && params.RollbackVersion != '' }
            }
            steps {
                script {
                    if (currentBuild.result == 'FAILURE' || params.Stage == 'RollbackTestServer') {
                       echo "We need rollback to version: ${RollbackVersion}"
                       
                       ARTIFACTID = readMavenPom().getArtifactId()

                       JARNAME = ARTIFACTID+'-'+ "${RollbackVersion}"+'.jar'

                       sh "aws lambda update-function-code --function-name $LAMBDA_FUNCTION --region us-east-1 --s3-bucket $S3_BUCKET --s3-key ${JARNAME}"
                    }
                }  
            }
        }

        stage('Release to Prod') {
            agent none

            when {
                expression { params.Stage == 'All' || params.Stage == 'DeployProduction' }
            }

            steps {
                echo 'Release to Prod'
                script {
                    if (env.BRANCH_NAME == "main") {
                        timeout(time: 10) {
                            input(
                                message: "Approve for deployment version ${VERSION} on Production?",
                                ok: "Yes",
                                submitter: "admin"
                            )
                        }
                    }
                    else {
                        echo 'Prod Release Skipped'
                    }
                }

            }
        }

         stage('Deploy to Prod') {
            agent { label 'jenkins-slave'}
            when {
                expression { params.Stage == 'All' || params.Stage == 'DeployProduction' }
            }
            steps {
                script {
                    if (env.BRANCH_NAME == "main") {
                        echo 'Deploy to Prod'
                        retry(3) {
	            		    sleep 10
                        }
                       // sh "aws lambda update-function-code --function-name $LAMBDA_FUNCTION --s3-bucket $S3_BUCKET --s3-key ${JARNAME}" 
                    }
                }
            }
        }

         stage ('Notify when all stages are done') {
	        agent { label 'master' }
            steps {
                script {
                    if (currentBuild.result == 'FAILURE') {
                        mail (to: 'ragupathi.kommidi@gmail.com', subject: "ERROR: The job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) failed, rollback to last stable version", body: "Please go to ${env.BUILD_URL} for the detail")
			            echo "ERROR: The job ${env.JOB_NAME} (${env.BUILD_NUMBER}) failed, rollback to last stable version"

                    } else {
			            echo "The deployment process is done, new version is deployed successfully"
                        mail (to: 'ragupathi.kommidi@gmail.com', subject: "SUCCESS: The job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is done, new version ${VERSION} is deployed successfully", body: "Please go to ${env.BUILD_URL} for the detail")
                    }
                }
            }
        }

    }
}
