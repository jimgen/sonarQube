#!/usr/bin/env groovy

pipeline {
	agent { label 'master' }
	stages {
		stage('Checkout') {
			steps {
				checkout scm
			}
		}
		stage('Build image') {
            environment {
                IMAGE_NAME = "sonarqube"
            }
			steps {
				sh "docker build -t $IMAGE_NAME ."
			}
		} 
		stage('Push image') {
			environment {
                ACCOUNT_ID = "196093915263"
				IMAGE_NAME = "sonarqube"
            }
			steps {
				sh "echo \"Docker Image Pushed\""
				script {
					docker.withRegistry("https://196093915263.dkr.ecr.ap-southeast-2.amazonaws.com", "ecr:ap-southeast-2:aws_creds") {
						docker.image("sonarqube").push('latest')
					}	
				}
				sh "echo \"Docker Image Pushed\""
			}
		}
		stage('Run instance')
		{
			environment {
				TASK_NAME = "sonarqube-taskdef"
				CLUSTER_NAME = "jim-test"
				SERVICE_NAME = "sonarqube-service"
            }
			steps { 
				withCredentials( [[ $class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws_creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' ]]) {
					 echo "Listing contents of an S3 bucket." 
					sh "export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID"
					sh "export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY"
					sh 'export AWS_DEFAULT_REGION=ap-southeast-2'

					sh 'sed -e "s;%BUILD_NUMBER%;${BUILD_NUMBER};g" $TASK_NAME.json >  $TASK_NAME-v${BUILD_NUMBER}.json'
					sh 'aws ecs register-task-definition --family $TASK_NAME --cli-input-json file://$TASK_NAME-v${BUILD_NUMBER}.json --region ap-southeast-2'
					sh '''	
						TASK_REVISION=`aws ecs describe-task-definition --task-definition $TASK_NAME --region ap-southeast-2 | jq .taskDefinition.revision`
						SERVICES=`aws ecs describe-services --services $SERVICE_NAME --cluster $CLUSTER_NAME --region ap-southeast-2 | jq '.services[] | length'`
						if [ -z $SERVICES ]; then 
							aws ecs create-service --cluster $CLUSTER_NAME --region ap-southeast-2 --service-name $SERVICE_NAME --task-definition $TASK_NAME:$TASK_REVISION --desired-count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-0d1fcce01ec92f30e
],securityGroups=[sg-074a210c92193561f],assignPublicIp=ENABLED}"
						else 
							aws ecs update-service --cluster $CLUSTER_NAME --service $SERVICE_NAME --task-definition $TASK_NAME:$TASK_REVISION --desired-count 1 --region ap-southeast-2
						fi
					'''
			} }  
			
		}
		stage('Clean up')
		{		
			steps {
				sh 'docker image prune -a -f'
				deleteDir()
				script {
					currentBuild.result = 'SUCCESS'
				}
			}
		}
	}
	post {
        failure {
            script {
                currentBuild.result = 'FAILURE'
            }
        }

        always {
				sh "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"Result ${currentBuild.result}\"}' https://hooks.slack.com/services/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"
        }
    }
}