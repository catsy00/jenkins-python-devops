pipeline {
    environment {
        DOMAIN='apps.ocp4.paas.local'
        PRJ="tjkim-jenkinstest-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        APP='python'
        GIT_URL='https://github.com/catsy00/jenkins-python-devops'
    }
    agent none
    stages {
        stage('create') {
            steps {
                script {
                    // Uncomment to get lots of debugging output
                    //openshift.logLevel(1)
                    openshift.withCluster() {
                        echo("Create project ${env.PRJ}") 
                        openshift.newProject("${env.PRJ}")
                        openshift.withProject("${env.PRJ}") {
                            echo('Grant to developer read access to the project')
                            openshift.raw('policy', 'add-role-to-user', 'view', 'developer')
                            echo("Create app ${env.APP}") 
                            echo("git url ${env.GIT_URL}")
                            openshift.newApp("${env.GIT_URL}#${env.BRANCH_NAME}", "--strategy docker", "--name ${env.APP}")
                        }
                    }
                }
            }
        }
        stage('build') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.PRJ}") {
                            def bc = openshift.selector('bc', "${env.APP}")
                            echo("Wait for build from bc ${env.APP} to finish") 
                            timeout(5) {
                                def builds = bc.related('builds').untilEach(1) {
                                    def phase = it.object().status.phase
                                    if (phase == "Failed" || phase == "Error" || phase == "Cancelled") {
                                        error 'OpenShift build failed or was cancelled'
                                    }
                                    return (phase == "Complete")
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.PRJ}") {
                            echo("Expose route for service ${env.APP}") 
                            // Default Jenkins settings to not allow to query properties of an object
                            // So we cannot query the widlcard domain of the ingress controller
                            // Nor the auto genereted host of a route
                            openshift.expose("svc/${env.APP}", "--hostname ${env.PRJ}.${env.DOMAIN}")
                            echo("Wait for deployment ${env.APP} to finish") 
                            timeout(5) {
                                openshift.selector('deployment', "${env.APP}").rollout().status()
                            }
                        }
                    }
                }
            }
        }
    }
}