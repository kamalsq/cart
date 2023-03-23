@Library('Jenkins-shared-library') _

def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]



pipeline {
  environment {
	environment()
  }

  options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  agent {
    kubernetes {
        label 'jenkinsrun'
        defaultContainer 'dind'
        yaml """
          apiVersion: v1
          kind: Pod
          spec:
            containers:
            - name: dind
              image: squareops/jenkins-build-agent:latest
              securityContext:
                privileged: true
              volumeMounts:
                - name: dind-storage
                  mountPath: /var/lib/docker
            volumes:
              - name: dind-storage
                emptyDir: {}
        """
        }
      }
  
  stages {
    stage ('Static code Analysis') {
      steps {
// NodeJS Scan
        script { 
            static-code-anlysis()
       }
      }
  }
  
    stage('Build Docker Image') {   
      agent {
        kubernetes {
          label 'kaniko'
          yaml """
          apiVersion: v1
          kind: Pod
          metadata:
            name: kaniko              
          spec:
            restartPolicy: Never
            containers:
            - name: kaniko
              image: gcr.io/kaniko-project/executor:debug
              command:
              - /busybox/cat
              tty: true 
          """
        }
      }  
      steps {
       container('kaniko'){
            script {
		build-docker-image()              
            }   
            stash includes: 'build/*.tar', name: 'image'          
        }
      }
    }
    stage('Scan Docker Image') {
      when {		
	    anyOf {
                        branch 'main';
                        branch 'development'
	    }            
	   }
      agent {
        kubernetes {           
            containerTemplate {
              name 'trivy'
              image 'aquasec/trivy:0.35.0'
              command 'sleep'
              args 'infinity'
            }
        }
      }
      options { skipDefaultCheckout() }
      steps {
        container('trivy') {
           script {
                scan-docker-image()
           }
         }
      } 
    }    

    stage('Push to ECR') {
      when {
                branch 'main'
            }
       agent {
          kubernetes { 
          yaml """
          apiVersion: v1
          kind: Pod
          metadata:
            name: kaniko              
          spec:
            restartPolicy: Never
            containers:
            - name: kaniko
              image: gcr.io/kaniko-project/executor:debug
              command:
              - /busybox/cat
              tty: true 
          """  
            }
        }
      //  options { skipDefaultCheckout() }
       steps {        
         container('kaniko') {
           script {
                push-to-ecr()                                  
            }
	        }
        }
      }
 
     stage('Asking for Deploy in prod') {
            //   when {
            //      equals(actual: env.gitlabBranch , expected: "prod")
            //  }
            when {
                branch 'main'
            }
             steps {
                 input "Do you want to Deploy in Production?"
             }
         }
 
     stage('Deploy') {
      when {
                branch 'main'
            }
       steps {    
          container('dind') {
            script {
                deploy()
        }
      }
     }
  }

      post {
        failure {
            slackSend message: 'Pipeline for ' + env.JOB_NAME + ' with Build Id - ' +  env.BUILD_ID + ' Failed at - ' +  env.last_started
        }

        success {
            slackSend message: 'Pipeline for ' + env.JOB_NAME + ' with Build Id - ' +  env.BUILD_ID + ' SUCCESSFUL'
        }
    }
}
