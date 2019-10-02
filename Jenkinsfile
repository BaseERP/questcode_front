podTemplate(
    name: 'questcode', 
    namespace: 'devops', 
    label: 'questcode', 
    containers:[
        containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'docker', livenessProbe: containerLivenessProbe(execArgs: '', failureThreshold: 0, initialDelaySeconds: 0, periodSeconds: 0, successThreshold: 0, timeoutSeconds: 0), name: 'docker-container', resourceLimitCpu: '', resourceLimitMemory: '', resourceRequestCpu: '', resourceRequestMemory: '', ttyEnabled: true),
        containerTemplate(args: 'cat', command: '/bin/sh -c', image: 'lachlanevenson/k8s-helm:v2.11.0',  name: 'helm-container',  ttyEnabled: true)
    ], 
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')] 
)

{
    //Start Pipeline
    node('questcode') {
        def REPOS
        def IMAGE_NAME = "frontend"
        def KUBE_NAMESPACE
        def IMAGE_VERSION 
        def ENVIRONMENT 
        def GIT_REPOS_URL = "https://github.com/BaseERP/questcode_front.git"
        def GIT_BRANCH 
        def CHARTMUSEUM_URL = "http://helm-chartmuseum:8080"
        def HELM_CHART_NAME = "questcode/frontend"
        def HELM_DEPLOY_NAME  

        stage('Checkout') {
            echo 'Iniciando Clone do repositorio'
            REPOS = checkout([$class: 'GitSCM', branches: [[name: '*/master'], [name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: GIT_REPOS_URL]]])
            GIT_BRANCH = RESPOS.GIT_BRANCH
            if(GIT_BRANCH.equals("master")) {
               KUBE_NAMESPACE = "prod"
               ENVIRONMENT = "production"
            } else if(GIT_BRANCH.equals("develop")) {
               KUBE_NAMESPACE = "staging"
               ENVIRONMENT = "staging"
            } else {
                echo "Não existe pipeline para a branch ${GIT_BRANCH}"
                exit 0
            }
            HELM_DEPLOY_NAME = KUBE_NAMESPACE + "-frontend" 
            IMAGE_VERSION = sh label: '', returnStdout: true, script: 'sh read-package-version.sh'
            IMAGE_VERSION =IMAGE_VERSION.trim()
        }
        stage('Package') {
            container('docker-container') {
               echo 'Iniciando o empacotamento com docker'
               withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                    sh "docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}"
                    sh "docker build -t ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION} . --build-arg NPM_ENV='${ENVIRONMENT}'"
                    sh "docker push ${DOCKER_HUB_USER}/${IMAGE_NAME}:${IMAGE_VERSION}"
               }

            }
        } 
        stage('Deploy') {
            //CLIENTE DO HELM
            container('helm-container') {
                echo 'Iniciando Deploy Helm'
                sh 'helm init --client-only'
                sh "helm repo add questcode ${CHARTMUSEUM_URL}"
                sh 'helm repo update'
                try{
                    //Fazer helm upgrade
                    sh "helm upgrade --namespace=${KUBE_NAMESPACE} ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION}"
                } catch(Exception e) {
                    //Fazer helm install
                    sh "helm install --namespace=${KUBE_NAMESPACE} --name ${HELM_DEPLOY_NAME} ${HELM_CHART_NAME} --set image.tag=${IMAGE_VERSION}"
                }
            }

        }
    }
}