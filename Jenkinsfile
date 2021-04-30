pipeline {
    agent any
    stages {
        stage('Build info') {
            steps {
              sh 'env'
            }
        }

        stage('Checkout sources') {
            steps {
                checkout changelog: false, poll: false,
                    scm: [$class: 'GitSCM', branches: [[name: "${env.GIT_BRANCH}"]],
                    doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                    userRemoteConfigs: [[url: "${env.GIT_URL}"]]]
            }
        }

        stage('Rebuild forked civicrm image') {
            steps {
                script {
                    if (env.BUILD_FORK_IMAGE.toBoolean()) {
                        // increase timeout
                        timeout(time: 60, unit: 'MINUTES') {
                            // consider changes in nc_image_fix/Dockerfile
                            openshift.withCluster() {
                                openshift.withProject(/*"${env.PROJECT_NAME}"*/) {
                                    def buildSelector = openshift.selector("bc", 'latest-fork')
                                    buildSelector.startBuild("--follow=true")
                                    /* Alternatively to "--follow=true":
                                        * Do some parallel tasks while building.
                                        * When needed to wait for the build again and show logs, do:
                                        * build.logs('-f') */
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Apply configuration update') {
            steps {
                script {
                    // consider changes in openproject.yaml
                    openshift.withCluster() {
                        openshift.withProject(/*"${env.PROJECT_NAME}"*/) {
                            def template = readFile 'civicrm.yaml'
                            def config = null
                            if (env.BUILD_FORK_IMAGE.toBoolean()) {
                                config = openshift.process(template,
                                  '-p', "PVC_SIZE=${env.PVC_SIZE}",
                                  '-p', "CIVICRM_HOST=${env.CIVICRM_HOST}",
                                  '-p', "DATABASE_SECRET=${env.DATABASE_SECRET}",
                                  '-p', "COMMUNITY_IMAGE_KIND=ImageStreamTag",
                                  '-p', "COMMUNITY_IMAGE_NAME=latest-fork",
                                  '-p', "COMMUNITY_IMAGE_TAG=${env.COMMUNITY_IMAGE_TAG}")
                            } else {
                                config = openshift.process(template,
                                  '-p', "PVC_SIZE=${env.PVC_SIZE}",
                                  '-p', "CIVICRM_HOST=${env.CIVICRM_HOST}",
                                  '-p', "DATABASE_SECRET=${env.DATABASE_SECRET}",
                                  '-p', "COMMUNITY_IMAGE_NAME=${env.COMMUNITY_IMAGE_NAME}",
                                  '-p', "COMMUNITY_IMAGE_TAG=${env.COMMUNITY_IMAGE_TAG}")
                            }
                            openshift.apply(config)
                        }
                    }
                }
            }
        }
      
        stage('Wait for rollout to complete') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(/*"${env.PROJECT_NAME}"*/) {
                            def templateName = 'latest'
                            /*def rm = openshift.selector("dc", templateName)
                              .rollout().latest()*/
                            timeout(10) {
                                openshift.selector("dc", templateName)
                                  .related('pods').untilEach(1) {
                                    return (it.object().status.phase == "Running")
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
