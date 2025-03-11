pipeline {
    agent any

    environment {
        REGISTRY = 'user10.azurecr.io'
        IMAGE_NAME = 'delivery'
        AKS_CLUSTER = 'user10-aks'
        RESOURCE_GROUP = 'user10-rsrcgrp'
        AKS_NAMESPACE = 'default'
        AZURE_CREDENTIALS_ID = 'Azure-Cred'
        TENANT_ID = 'ecd8d459-73d3-48f6-bf87-629631dc2d71' // Service Principal 등록 후 생성된 ID
        GIT_USERNAME = 'rubycake-skeptic'
        GIT_USER_EMAIL = 'ruby-cake-skeptic@duck.com'
        GITHUB_CREDENTIALS_ID = 'Github-Cred'
        GITHUB_REPO = 'github.com/rubycake-skeptic/reqres_delivery.git'
        GITHUB_BRANCH = 'master'
    }
 
    stages {

        stage('Check Modified Files') {
            steps {
                script {
                    // Git 변경된 파일 목록 가져오기
                    def changedFiles = sh(returnStdout: true, script: 'git diff --name-only HEAD~1 HEAD').trim().split('\n')

                    // 'src/' 폴더 변경 여부 확인
                    echo | ls
                    def targetFolder = 'src/*'
                    def isModified = changedFiles.any { it.startsWith(targetFolder) }
    
                    if (isModified) {
                        echo "Changes detected in ${targetFolder}. Proceeding with the pipeline."
                    } else {
                        echo "No changes in ${targetFolder}. Stopping pipeline execution."
                        currentBuild.result = 'NOT_BUILT'
                        return
                    }
                }
            }
        }
        
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }
        
        stage('Maven Build') {
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Azure Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: env.AZURE_CREDENTIALS_ID, usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                        sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant ${TENANT_ID}'
                    }
                }
            }
        }
        
        stage('Push to ACR') {
            steps {
                script {
                    sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('CleanUp Images') {
            steps {
                sh """
                docker rmi ${REGISTRY}/${IMAGE_NAME}:v$BUILD_NUMBER
                """
            }
        }
        
        // stage('Deploy to AKS') {
        //     steps {
        //         script {
        //             sh "az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${AKS_CLUSTER}"
        //             sh """
        //             sed 's/latest/v${env.BUILD_ID}/g' azure/deploy.yaml > output.yaml
        //             cat output.yaml
        //             kubectl apply -f output.yaml
        //             kubectl apply -f azure/service.yaml
        //             rm output.yaml
        //             """
        //         }
        //     }
        // }

        stage('Update deploy.yaml') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    sh """
                    sed -i 's|image: ${REGISTRY}/${IMAGE_NAME}:.*|image: ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_ID}|' azure/deploy.yaml
                    cat azure/deploy.yaml
                    """
                }
            }
        }
        
        stage('Commit and Push to GitHub') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GITHUB_CREDENTIALS_ID, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                            rm -rf repo
                            git config --global user.email "${GIT_USER_EMAIL}"
                            git config --global user.name "Jenkins CI"
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${GITHUB_REPO} repo
                            cp azure/deploy.yaml repo/azure/deploy.yaml
                            cd repo
                            git add azure/deploy.yaml
                            git commit -m "Update deploy.yaml with build ${env.BUILD_NUMBER}"
                            git push origin ${GITHUB_BRANCH}
                            cd ..
                            rm -rf repo
                        """
                    }
                }
            }
        } 
    }
}
