pipeline {
    /* insert Declarative Pipeline here */

   agent {
      node {label 'python'}
    }

   environment {
    APPLICATION_NAME = 'python-nginx'
    GIT_REPO="http://github.com/ruddra/openshift-python-nginx.git"
    GIT_BRANCH="master"
    STAGE_TAG = "promoteToQA"
    DEV_PROJECT = "dev"
    STAGE_PROJECT = "stage"
    TEMPLATE_NAME = "python-nginx"
    ARTIFACT_FOLDER = "target"
    PORT = 8081;
   }

  stages 
  {
          stage("Stage One - Get Latest Code")
          {
                      // Do Something
              steps { git branch: "${GIT_BRANCH}", url: "${GIT_REPO}" // declared in environment 
                    }
          }
    stage("Install Dependencies") 
     {
           steps {
                 sh """
                 pip install virtualenv
                 virtualenv --no-site-packages .
                 source bin/activate
                 pip install -r app/requirements.pip
                 deactivate
                 """
                }
     }

    stage('Run Tests') 
     {
        steps {
        sh '''
        source bin/activate
        nosetests app --with-xunit
        deactivate
        '''
        junit "nosetests.xml"
        }
     }

    stage('Store Artifact')
    {
       steps {
              script {
                 def safeBuildName  = "${APPLICATION_NAME}_${BUILD_NUMBER}",
                 artifactFolder = "${ARTIFACT_FOLDER}",
                 fullFileName   = "${safeBuildName}.tar.gz",
                 applicationZip = "${artifactFolder}/${fullFileName}"
                 applicationDir = ["app",
                              "config",
                              "Dockerfile",
                              ].join(" ");
                 def needTargetPath = !fileExists("${artifactFolder}")
                 if (needTargetPath) {
                  sh "mkdir ${artifactFolder}"
                  }
                sh "tar -czvf ${applicationZip} ${applicationDir}"
                archiveArtifacts artifacts: "${applicationZip}", excludes: null,               onlyIfSuccessful: true
              }
            }
     }

     
     stage('Create Image Builder') 
   {
       when {
        expression {
            openshift.withCluster() {
            openshift.withProject(DEV_PROJECT) {
                return !openshift.selector("bc", "${TEMPLATE_NAME}").exists();
                }
            }
        }
       }
       steps {
             script {
                openshift.withCluster() {
                     openshift.withProject(DEV_PROJECT) {
                         openshift.newBuild("--name=${TEMPLATE_NAME}", "--docker-image=docker.io/nginx:mainline-alpine", "--binary=true")
                     }
                }
             }
       }
   } 


     stage('Build Image') 
    {
       steps {
              script {
                   openshift.withCluster() {
                       openshift.withProject(env.DEV_PROJECT) {
                             openshift.selector("bc", "$TEMPLATE_NAME").startBuild("--from-archive=${ARTIFACT_FOLDER}/${APPLICATION_NAME}_${BUILD_NUMBER}.tar.gz", "--wait=true")
                       }
                   }
              }
       }
    }



      stage('Deploy to DEV') 
     {
           when {
                  expression {
                        openshift.withCluster() {
                                  openshift.withProject(env.DEV_PROJECT) {
                                     return !openshift.selector('dc', "${TEMPLATE_NAME}").exists()
                                  }
                         }
                   }
                 }
          steps {
                 script {
                    openshift.withCluster() {
                                 openshift.withProject(env.DEV_PROJECT) {
                                       def app = openshift.newApp("${TEMPLATE_NAME}:latest")
                                       app.narrow("svc").expose("--port=${PORT}");
                                       def dc = openshift.selector("dc", "${TEMPLATE_NAME}")
                                       while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                                       sleep 10
                                       }
                                 }
                    }
                 }
         }
     }


    stage('Promote to STAGE?') 
    {
      steps {
         timeout(time:15, unit:'MINUTES') {
           input message: "Promote to STAGE?", ok: "Promote"
          }
        script {
          openshift.withCluster() {
          openshift.tag("${DEV_PROJECT}/${TEMPLATE_NAME}:latest", "${STAGE_PROJECT}/${TEMPLATE_NAME}:${STAGE_TAG}")
           }
        }
      }
    }



    stage('Rollout to STAGE') 
   {
     steps {
        script {
            openshift.withCluster() {
              openshift.withProject(STAGE_PROJECT) {
                  if (openshift.selector('dc', '${TEMPLATE_NAME}').exists()) {
                      openshift.selector('dc', '${TEMPLATE_NAME}').delete()
                      openshift.selector('svc', '${TEMPLATE_NAME}').delete()
                       openshift.selector('route', '${TEMPLATE_NAME}').delete()
                }
            openshift.newApp("${TEMPLATE_NAME}:${STAGE_TAG}").narrow("svc").expose("--port=${PORT}")
            }
         }
      }
    }
   }


   stage('Scale in STAGE') 
  {
    steps {
      script {
          openshiftScale(namespace: "${STAGE_PROJECT}", deploymentConfig: "${TEMPLATE_NAME}", replicaCount: '3')
        }
    }
   }




 }// Stages END

}
