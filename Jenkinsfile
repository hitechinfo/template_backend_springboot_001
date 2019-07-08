@Library('retort-lib') _ 
def label = "jenkins-${UUID.randomUUID().toString()}" 

def ZCP_USERID='mgmtsv-admin' 
def DOCKER_IMAGE='template/springboot' 
def K8S_NAMESPACE='submodule-template' 

podTemplate(label:label, 
    serviceAccount: "zcp-system-sa-${ZCP_USERID}", 
     containers: [ 
     containers: [ 
         containerTemplate(name: 'maven', image:'maven:3-jdk-12', ttyEnabled: true, command: 'cat'), 
         containerTemplate(name: 'docker', image:'17-dind', ttyEnabled: true, command: 'dockerd-entrypoint.sh', privileged: true), 
         containerTemplate(name: 'kubectl', image:'lachlanevenson/k8s-kubectl:v1.13.6', ttyEnabled: true, command: 'cat') 
     ], 
     volumes: [ 
         persistentVolumeClaim(mountPath: '/root/.m2', claimName: 'zcp-jenkins-mvn-repo') 
     ]) { 
     node(label) { 
 
         stage('SOURCE CHECKOUT') { 	
             def repo = checkout scm 
             env.SCM_INFO = repo.inspect() 
         } 
          
          
 		stage('BUILD') { 
 			container('maven') { 
 				mavenBuild goal: 'clean package', systemProperties:['maven.repo.local':"/root/.m2/${JOB_NAME}"] 
 			} 
 		} 
          
         stage('BUILD DOCKER IMAGE') { 
             container('docker') { 
                 dockerCmd.build tag: "${HARBOR_REGISTRY}/${DOCKER_IMAGE}:${BUILD_NUMBER}" 
                 dockerCmd.push registry: HARBOR_REGISTRY, imageName: DOCKER_IMAGE, imageVersion: BUILD_NUMBER, credentialsId: "HARBOR_CREDENTIALS" 
             } 
         } 

         stage('DEPLOY') { 
             container('kubectl') {
                 kubeCmd.apply file: 'k8s/template-ingress.yaml', namespace:K8S_NAMESPACE
                 kubeCmd.apply file: 'k8s/template-service.yaml', namespace:K8S_NAMESPACE  
                 yaml.update file: 'k8s/template-deployment.yaml', update: ['.spec.template.spec.containers[0].image': "${HARBOR_REGISTRY}/${DOCKER_IMAGE}:${BUILD_NUMBER}"] 
                 kubeCmd.apply file: 'k8s/template-deployment.yaml', wait: 300, recoverOnFail: false, namespace: K8S_NAMESPACE 
                  
             } 
         } 
          
     } 
 }