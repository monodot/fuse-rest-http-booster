openshift.withCluster() {
  env.NAMESPACE = openshift.project()
  env.POM_FILE = env.BUILD_CONTEXT_DIR ? "${env.BUILD_CONTEXT_DIR}/pom.xml" : "pom.xml"
  env.APP_NAME = "${JOB_NAME}".replaceAll(/-build.*/, '')
  echo "Starting Pipeline for ${APP_NAME}..."
  env.BUILD = "${env.NAMESPACE}"
  env.DEV = "${APP_NAME}-dev"
  env.STAGE = "${APP_NAME}-stage"
  env.PROD = "${APP_NAME}-prod"
}

pipeline {
  // Use Jenkins Maven slave
  // Jenkins will dynamically provision this as OpenShift Pod
  // All the stages and steps of this Pipeline will be executed on this Pod
  // After Pipeline completes the Pod is killed so every run will have clean
  // workspace
  agent {
    label 'maven'
  }
  
  stages {
    
    stage('Git Checkout') {
      steps {
        // Turn off Git's SSL cert check, uncomment if needed
        // sh 'git config --global http.sslVerify false'
        git url: "${APPLICATION_SOURCE_REPO}"
      }
    }
    
    stage('Deploy OCP Objects to DEV') {
      agent {
        label 'jenkins-slave-ansible'
      }
      
      steps {
        echo '👷 Create OpenShift objects using openshift-applier...'
        sh  '''
        ansible-galaxy install -r requirements.yml --roles-path=.applier/roles
        ansible-playbook -i .applier/inventory .applier/apply.yml -e filter_tags=dev -e sb_dev_namespace=${NAMESPACE_DEV}
        '''
      }
    }


        stage('Build') {
            agent {
                node {
                    label "jenkins-slave-mvn"
                }
            }
            steps {
                echo '❤️ Running Maven build...'
                sh "mvn -B clean install -DskipTests=true"

                echo '💚 Running tests...'
                sh "mvn -B test"

                echo '💙 Deploying to artifact repository...'
                sh "mvn -B deploy -DskipTests=true"
            }

            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }

        stage('Bake') {
            agent {
                node {
                    label "jenkins-slave-mvn"
                }
            }
            steps {
                script {
                    env.MAVEN_DEPLOY_REPO = "labs-snapshots"
                    pom = readMavenPom file: 'pom.xml'
                    env.POM_GROUP_ID = pom.groupId
                    env.POM_VERSION = pom.version
                    env.POM_ARTIFACT_ID = pom.artifactId
                }
                echo '### Downloading artifact at version ${POM_VERSION}...'
                
                sh "mvn dependency:get -DremoteRepositories=http://nexus-labs-ci-cd.apps.boeing.emea-1.rht-labs.com/repository/${MAVEN_DEPLOY_REPO} -DgroupId=${POM_GROUP_ID} -DartifactId=${POM_ARTIFACT_ID} -Dversion=${POM_VERSION} -Dtransitive=false"
                sh "mvn dependency:copy -Dartifact=${POM_GROUP_ID}:${POM_ARTIFACT_ID}:${POM_VERSION} -DoutputDirectory=."
                sh "mv ${POM_ARTIFACT_ID}-${POM_VERSION}.jar application.jar"


                echo '🎁 Create Linux container image from package'
                sh  '''
                       oc patch bc ${APP_NAME} -p "{\\"spec\\":{\\"output\\":{\\"to\\":{\\"kind\\":\\"ImageStreamTag\\",\\"name\\":\\"${APP_NAME}:${JENKINS_TAG}\\"}}}}" -n ${BUILD_NAMESPACE}
                       oc start-build ${APP_NAME} --from-file=application.jar --follow -n ${BUILD_NAMESPACE}
                   '''
            }
            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }

        stage('Deploy Application to Dev') {
            agent {
                node {
                    label 'master'
                }
            }
            steps {
                echo '🍒 Set env vars and image for deployment'
                sh '''
                    oc set image dc/${APP_NAME} ${APP_NAME}=docker-registry.default.svc:5000/${DEV_NAMESPACE}/${APP_NAME}:${JENKINS_TAG} -n ${DEV_NAMESPACE}
                    oc rollout latest dc/${APP_NAME} -n ${DEV_NAMESPACE}
                '''
            }
            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }

        stage("Verify aplication in DEV") {
            agent {
                node { 
                    label "master"
                }
            }
            steps {
                echo 'Verify Deployment in RELEASE'
                openshiftVerifyDeployment depCfg: "${env.APP_NAME}", 
                    namespace: env.DEV_NAMESPACE, 
                    replicaCount: '1', 
                    verbose: 'false', 
                    verifyReplicaCount: 'true', 
                    waitTime: '',
                    waitUnit: 'sec'
                
            }
            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }


        stage('Integration Test') {
            agent {
                node {
                    label "jenkins-slave-mvn"
                }
            }
            steps {
                echo '🤓 Skipping Kubernetes integration tests...'
/*
                sh "mvn -B clean test -Dtest=*KT"
*/
            }

            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }


        /*
        stage('Manual Approval to promote to RELEASE') {
            agent none
            steps {
                script {
                    slackSend (color: '#0000FF', message: "Jenkins APPROVAL required")
                    slackSend (color: '#0000FF', message: "URL to APPROVE or DISCARD:  (${env.BUILD_URL}input)")
                    env.PROMOTE_TO_RELEASE = input message: 'User Approval Required',
                    parameters: [choice(name: 'Promote to RELEASE', choices: 'no\nyes', description: 'There are new DEV Images. Do you want promote them to RELEASE?')]
                }
            }
        }
        */

         stage("Promote to RELEASE") {
            agent {
                node { 
                    label "master"
                }
            }
            /*
            when {
                environment name: 'PROMOTE_TO_RELEASE', value: 'yes'
            }
            */
            steps {
                    echo 'Promoting Container Image to Release'
                    script {
                        openshift.withProject( "${DEV_NAMESPACE}" ) {
                            echo "Promoting to ${RELEASE_NAMESPACE}"
                            openshift.tag("${APP_NAME}:${JENKINS_TAG}", "${RELEASE_NAMESPACE}/${APP_NAME}:${RELEASE_TAG}")
                        }
                    }
            }
            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }

        stage("Deploy OCP Objects to RELEASE") {
            agent {
                node { 
                    label "jenkins-slave-ansible"
                }
            }
            steps {
                echo '👷 Create OpenShift objects using openshift-applier...'

                sh  '''
                ansible-galaxy install -r .applier/requirements.yml --roles-path=.applier/roles
                ansible-playbook -i .applier/inventory .applier/apply.yml -e filter_tags=release
                '''
            }

            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }

        stage('Deploy Application to Release') {
            agent {
                node {
                    label 'master'
                }
            }
            steps {
                echo '🍒 Set env vars and image for deployment'
                sh '''
                    oc set image dc/${APP_NAME} ${APP_NAME}=docker-registry.default.svc:5000/${RELEASE_NAMESPACE}/${APP_NAME}:${RELEASE_TAG} -n ${RELEASE_NAMESPACE}
                    oc rollout latest dc/${APP_NAME} -n ${RELEASE_NAMESPACE}                    
                '''
            }
            post {
                failure {
                    notifyBuild('FAIL')
                }
            }
        }

        stage("Verify Deployment in RELEASE") {
            agent {
                node {
                    label "master"
                }
            }
            steps {
                echo 'Verify Deployment in RELEASE'
                openshiftVerifyDeployment depCfg: "${env.APP_NAME}", 
                    namespace: env.RELEASE_NAMESPACE, 
                    replicaCount: '1', 
                    verbose: 'false', 
                    verifyReplicaCount: 'true', 
                    waitTime: '',
                    waitUnit: 'sec'
            }
            post {
                failure {
                    notifyBuild('FAIL')
                }
                success {
                     // Merge changes into release branch
                    git branch: 'master',
                        credentialsId: 'labs-ci-cd-gitlab-secret',
                        url: "$GIT_URL"
                      
                    sh  '''
                        GIT_SERVER_NAME=$(echo ${GIT_URL} | cut -d'/' -f3)
                        GIT_PROJECT_NAME=$(echo ${GIT_URL} | cut -d'/' -f4)
                        GIT_REPO_NAME=$(echo ${GIT_URL} | cut -d'/' -f5)

                        git remote remove origin
                        git remote add origin https://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@${GIT_SERVER_NAME}/${GIT_PROJECT_NAME}/${GIT_REPO_NAME}
                        
                        git config user.name 'JenkinsCI'
                        git config user.email 'jenkins@red-hat-labs-boeing.com'

                        git fetch
                        git checkout release
                        git pull origin release

                        # Eval if there is any difference
                        RELEASE_COMMIT=$(git rev-parse --verify HEAD)
                        git diff $GIT_COMMIT $RELEASE_COMMIT > git_diff.out
                        if [[ -s git_diff.out ]]; then
                            # File size greater than zero, there are changes
                            git checkout master
                            git pull origin master
                            git push origin master:release -f
                           
                        else
                            # File size equal to zero, no changes
                            echo "Nothing to commit -- MASTER and RELEASE branches are up to date"
                        fi

                    '''
                    notifyBuild('SUCCESSFUL')
                }
            }
        }

    }
}
