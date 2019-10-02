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
        def IMAGE_VERSION 
        def ENVIRONMENT = "staging"
        def GIT_REPOS_URL = "https://github.com/BaseERP/questcode_front.git"
        def CHARTMUSEUM_URL = "http://helm-chartmuseum:8080"

        stage('Checkout') {
            echo 'Iniciando Clone do repositorio'
            REPOS = checkout([$class: 'GitSCM', branches: [[name: '*/master'], [name: '*/develop']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: GIT_REPOS_URL]]])
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
                sh "helm upgrade staging-frontend questcode/frontend --set image.tag=${IMAGE_VERSION}"
            }

        }
    }
}