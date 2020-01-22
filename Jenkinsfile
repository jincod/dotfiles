#!/usr/bin/env groovy

def slackChannel = '#dev-channel'
def PROJECT_VERSION = "1.0.${BUILD_NUMBER}"
def PROJECT_NAME = "PRJ"
def SONAR_PROJECT_KEY = "prj"
def VAULT_URL = "<vault_url>"
def api
def CURRENT_ENV, REGISTRY_URL

switch(env.BRANCH_NAME) {
	case 'master':
		CURRENT_ENV = 'prod'
		REGISTRY_URL = '<prod.registry.url>'
		break
	case 'uat':
		CURRENT_ENV = 'uat'
		REGISTRY_URL = '<uat.registry.url>'
		break
	case 'develop':
		CURRENT_ENV = 'dev'
		REGISTRY_URL = '<dev.registry.url>'
		break
}


pipeline {
	agent {
		label 'linux'
	}
	options {
		disableConcurrentBuilds()
		buildDiscarder(logRotator(numToKeepStr: '5'))
	}

	stages {
		stage('Tests') {
			steps {
				sh "dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover ./src/Tests/Tests.csproj --logger \"junit;LogFileName=result.xml\""
				junit 'src/Tests/TestResults/TestResults.xml'
			}
		}

		stage('Build Docker images') {
			when {
				anyOf {
					branch 'develop';
					branch 'uat';
					branch 'master';
				}
			}
			steps {
				script {
					api = docker.build("${CURRENT_ENV}/api", "-f ./src/Api/Dockerfile .")
				}
			}
		}

		stage ('Publish docker images') {
			when {
				anyOf {
					branch 'develop';
					branch 'uat';
					branch 'master';
				}
			}
			steps {
				script {
					docker.withRegistry("https://${REGISTRY_URL}", "prj-docker-${CURRENT_ENV}") {
						api.push("${PROJECT_VERSION}")
						api.push("latest")
						// Cleaning
						sh "docker rmi ${REGISTRY_URL}/${CURRENT_ENV}/api:${PROJECT_VERSION} || :"
					}
				}
			}
		}

		stage('SonarQube analysis') {
			when {
				anyOf {
					branch 'develop';
				}
			}
			steps {
				script {
					scannerHome = tool 'Scanner .Net Core';
				}

				withSonarQubeEnv('SonarQube Server') {
					sh "dotnet ${scannerHome}/SonarScanner.MSBuild.dll begin /k:${SONAR_PROJECT_KEY} /n:${PROJECT_NAME} /v:${PROJECT_VERSION} /d:sonar.host.url=${SONAR_HOST_URL} /d:sonar.login=${SONAR_AUTH_TOKEN} /d:sonar.sources=src /d:sonar.javascript.file.suffixes=.js,.jsx /d:sonar.sourceEncoding=UTF-8"
					sh "dotnet build"
					sh "dotnet ${scannerHome}/SonarScanner.MSBuild.dll end /d:sonar.login=${SONAR_AUTH_TOKEN}"
				}
			}
		}
	}

	post {
		failure {
			slackSend channel: slackChannel, color: 'danger', message: "Error during build: $PROJECT_NAME $BRANCH_NAME <${BUILD_URL}|${BUILD_NUMBER}>"
		}
		fixed {
			slackSend channel: slackChannel, color: 'good', message: "Fixed: $PROJECT_NAME $BRANCH_NAME <${BUILD_URL}|${BUILD_NUMBER}>"
		}
	}
}
