pipeline {
    agent { label 'PROCX' }
    options {
        timeout(time: 1, unit: 'HOURS')
    }
    environment{
    SERVICE = "reachx-galaxy-dev"
    BASE_PATH = "/home/devops/procx_root/workspace/Reachx_galaxy_development"
    ENV_PATH ="kv/projects/reachx/galaxy"
    SERVICE_ENV=".env"
    DOCKER_TAG = "${env.BUILD_ID}"
    HELM_DIR = "reachx-galaxy-chart"
    HELM_BRANCH="reachx-galaxy-dev"
    IMAGE_TAG = "${BUILD_NUMBER}"+"-"+"${LATEST_COMMIT_SHORT}"
    LATEST_COMMIT_FULL = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    LATEST_COMMIT_SHORT = LATEST_COMMIT_FULL.take(7)
    }
    parameters {
        string(name: 'SCM_URL', defaultValue: 'https://github.com/DriveX-Mobility-Private-Limited/galaxy.git', description: 'SCM URL for source code')
        choice(name: 'BRANCH', choices: 'Team')
    }
    stages {     
        stage('Clean') {
            steps {
                cleanWs()
            }
        }              
        stage('SCM') {         
            steps {   
                    git(
                        url: "${params.SCM_URL}",
                        branch: "${params.BRANCH}",
                        credentialsId: "Procx-github",
                    )
            } 
        }
        stage('secrets from vault') { 
            steps {
                withVault(configuration: [engineVersion: 1, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'ENV_HOST'], [vaultKey: 'APP_ENV']]]]) {
                    sh "vault kv get -address=${ENV_HOST} -field=\"${BRANCH}\" ${ENV_PATH}/ > ${BASE_PATH}/${APP_ENV}"
                }
            }
        }
        stage('Fetch-git-token') { 
            steps {
                withVault(configuration: [engineVersion: 1, failIfNotFound: false, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[engineVersion: 1, path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'GIT_TOKEN']]]]) {
                    sh "sed -i 's|{GITHUB_TOKEN}|${GIT_TOKEN}|g' requirements.txt"
                    sh "cd ${BASE_PATH}"
                    sh "cat requirements.txt"
                } 
            }   
        }                   
        stage('Build docker-image') { 
            steps {
                withVault(configuration: [engineVersion: 1, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'DOCKERHUB_CREDENTIALS_USR']]]]) {
                    sh 'cd ${BASE_PATH}'
                    sh 'docker image build -t $DOCKERHUB_CREDENTIALS_USR/$SERVICE:${IMAGE_TAG}  ${BASE_PATH}'
                }  
            }   
        } 
        stage('Dockerhub login') {         
            steps {
                withVault(configuration: [engineVersion: 1, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[engineVersion: 1, path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'DOCKERHUB_CREDENTIALS_USR'], [vaultKey: 'DOCKERHUB_CREDENTIALS_PW']]]]) {
                    sh 'docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PW'
                }
            }
        }
        stage('Push docker image') {        
            steps {
                withVault(configuration: [engineVersion: 1, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[engineVersion: 1, path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'DOCKERHUB_CREDENTIALS_USR'], [vaultKey: 'DOCKERHUB_CREDENTIALS_PW']]]]) {
                sh 'docker push $DOCKERHUB_CREDENTIALS_USR/$SERVICE:${IMAGE_TAG}'
                sh 'docker rmi $DOCKERHUB_CREDENTIALS_USR/$SERVICE:"$IMAGE_TAG"'
               }
            }
        }
        stage('Clone chart') {
            steps {
                withVault(configuration: [engineVersion: 1, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'HELM_REPO']]]]) {
                    dir('reachx-galaxy-chart') {
                        git(url: "${HELM_REPO}", branch: "${HELM_BRANCH}", credentialsId: "Procx-github")
                    }
                }
            }
        }
        stage('Update the chart') { 
            steps {
                withVault(configuration: [engineVersion: 1, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[engineVersion: 1, path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'GIT_USER'], [vaultKey: 'GIT_TOKEN']]]]) {
                    sh "cd ${BASE_PATH}/${HELM_DIR} && pwd && ls && sed -E -i 's/appVersion: .+/appVersion: $IMAGE_TAG/' Chart.yaml && git add Chart.yaml && git status && git commit Chart.yaml -m 'Updated the Charts yaml with version $IMAGE_TAG file' && git push https://${GIT_USER}:${GIT_TOKEN}@github.com/DriveX-Mobility-Private-Limited/drivex-deployments.git"
                } 
            }   
        }
        stage('sync argocd app') {
            steps {
                withVault(configuration: [engineVersion: 1, skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[path: 'kv/projects/cicd-pipelines-credentials', secretValues: [[vaultKey: 'ARGOCD_SERVER'], [vaultKey: 'ARGOCD_USERNAME'], [vaultKey: 'ARGOCD_PASSWORD']]]]) {
                sh ('argocd login --grpc-web $ARGOCD_SERVER --username $ARGOCD_USERNAME --password $ARGOCD_PASSWORD')    
                }
                sh "argocd app sync $SERVICE"
                }                  
        }
        stage('Waiting for ArgoCD App Status'){
            steps {
                script {
                def retries = 0
                def maxRetries = 10
                def healthStatus = 'Progressing'

                while (healthStatus.trim() == 'Progressing' && retries < maxRetries) {
                    def appJson = sh(returnStdout: true, script: "argocd app get ${SERVICE} -o=json").trim()
                    def json = readJSON text: appJson

                    healthStatus = json.status.health.status.trim()

                    if (healthStatus == 'Progressing') {
                        echo 'Health status is Processing. Waiting for the application to become Healthy...'
                        sleep time: 5, unit: 'SECONDS'
                        retries++
                        }
                    }

                if (healthStatus == 'Healthy') {
                    echo 'Health status is Healthy. Continuing with the pipeline.'
                    } else {
                        currentBuild.result = 'FAILURE'
                        error "Health status is ${healthStatus}. Build failed."
                        }
                    }
                }
        }
    }                      
    post {
        success {
            withVault(configuration: [engineVersion: 1, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'Deployment_updates']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'Success']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'Failure']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'reachx_galaxy_dev_success']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'reachx_galaxy_dev_failure']]]]) {
                googlechatnotification message:"$reachx_galaxy_dev_success", notifySuccess: "$Success", url: "$Deployment_updates"
                }
        }
        failure {
            withVault(configuration: [engineVersion: 1, timeout: 60, vaultCredentialId: 'jenkins-role', vaultUrl: 'https://vault.drivex.co.in'], vaultSecrets: [[path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'Deployment_updates']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'Success']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'Failure']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'reachx_galaxy_dev_success']]], [path: 'kv/projects/procx/google-notification', secretValues: [[vaultKey: 'reachx_galaxy_dev_failure']]]]) {
                googlechatnotification message:"$reachx_galaxy_dev_failure", notifySuccess: "$Failure", url: "$Deployment_updates"
                }
        }            
    }                    
}
