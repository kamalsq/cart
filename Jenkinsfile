def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]



pipeline {
  environment {
    doError = '0'
    AWS_ACCOUNT_ID = 271251951598
    AWS_REGION = "us-east-2"
    DOCKER_REPO_BASE_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    DOCKER_REPO_NAME = """${sh(
                returnStdout: true,
                script: 'basename=$(basename $GIT_URL) && echo ${basename%.*}'
            ).trim()}"""
    HELM_CHART_GIT_REPO_URL = "https://gitlab.com/sq-ia/ref/msa-app/helm.git"
    HELM_CHART_GIT_BRANCH = "qa"
    GIT_USER_EMAIL = "pipelines@squareops.com"
    GIT_USER_NAME = "squareops"  
    DEPLOYMENT_STAGE = """${sh(
                returnStdout: true,
                script: 'echo ${GIT_BRANCH#origin/}'
            ).trim()}"""
    last_started_build_stage = ""   
    IMAGE_NAME="${DOCKER_REPO_BASE_URL}/${DOCKER_REPO_NAME}/stg"
    def BUILD_DATE = sh(script: "echo `date +%d_%m_%Y`", returnStdout: true).trim();
    def scannerHome = tool 'SonarqubeScanner';
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
              last_started = env.STAGE_NAME
              echo 'Build start'              
              sh '''/kaniko/executor --dockerfile Dockerfile  --context=`pwd` --destination=${IMAGE_NAME}:${BUILD_NUMBER}-${BUILD_DATE} --no-push --oci-layout-path `pwd`/build/ --tarPath `pwd`/build/${DOCKER_REPO_NAME}-${BUILD_NUMBER}.tar
              '''               
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
                askfordeploy()
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
