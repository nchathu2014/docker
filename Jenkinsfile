#!/usr/bin/env groovy

def listOfProperties = []
listOfProperties << buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '5'))

// Only master branch will run on a timer basis
if (env.BRANCH_NAME.trim() == 'master') {
    listOfProperties << pipelineTriggers([cron('''H H/6 * * 0-2,4-6
H 6,21 * * 3''')])
}

properties(listOfProperties)

stage('Build') {
    def builds = [:]
    builds['windows'] = {
        nodeWithTimeout('windock') {
            stage('Checkout') {
                checkout scm
            }

            if (!infra.isTrusted()) {

                /* Outside of the trusted.ci environment, we're building and testing
                * the Dockerfile in this repository, but not publishing to docker hub
                */
                stage('Build') {
                    infra.withDockerCredentials {
                        powershell './make.ps1'
                    }
                }

                stage('Test') {
                    infra.withDockerCredentials {
                        def windowsTestStatus = powershell(script: './make.ps1 test', returnStatus: true)
                        junit(allowEmptyResults: true, keepLongStdio: true, testResults: 'target/**/junit-results.xml')
                        if (windowsTestStatus > 0) {
                            // If something bad happened let's clean up the docker images
                            powershell(script: '& docker system prune --force --all', returnStatus: true)
                            error('Windows test stage failed.')
                        }
                    }
                }

                // disable until we get the parallel changes merged in
                //def branchName = "${env.BRANCH_NAME}"
                //if (branchName ==~ 'master'){
                //    stage('Publish Experimental') {
                //        infra.withDockerCredentials {
                //            withEnv(['DOCKERHUB_ORGANISATION=jenkins4eval','DOCKERHUB_REPO=jenkins']) {
                //                powershell './make.ps1 publish'
                //            }
                //        }
                //    }
                //}

                // Let's always clean up the docker images at the very end
                powershell(script: '& docker system prune --force --all', returnStatus: true)
            } else {
                /* In our trusted.ci environment we only want to be publishing our
                * containers from artifacts
                */
                stage('Publish') {
                    infra.withDockerCredentials {
                        withEnv(['DOCKERHUB_ORGANISATION=jenkins','DOCKERHUB_REPO=jenkins']) {
                            powershell './make.ps1 publish'
                        }
                    }
                }
            }
        }
    }

    def images = ['almalinux_jdk11', 'alpine_jdk8', 'centos7_jdk8', 'centos8_jdk8', 'debian_jdk11', 'debian_jdk8', 'debian_slim_jdk8', 'rhel_ubi8_jdk11']
    for (i in images) {
        def imageToBuild = i

        builds[imageToBuild] = {
            nodeWithTimeout('docker && aws') {
                deleteDir()

                stage('Checkout') {
                    checkout scm
                }

                if (!infra.isTrusted()) {
                    stage('shellcheck') {
                        sh 'make shellcheck'
                    }

                    /* Outside of the trusted.ci environment, we're building and testing
                    * the Dockerfile in this repository, but not publishing to docker hub
                    */
                    stage("Build linux-${imageToBuild}") {
                        infra.withDockerCredentials {
                            sh "make build-${imageToBuild}"
                        }
                    }

                    stage("Test linux-${imageToBuild}") {
                        sh "make prepare-test"
                        try {
                            infra.withDockerCredentials {
                                sh "make test-${imageToBuild}"
                            }
                        } catch(err) {
                            error("${err.toString()}")
                        } finally {
                            junit(allowEmptyResults: true, keepLongStdio: true, testResults: 'target/*.xml')
                        }
                    }

                } else {
                    /* In our trusted.ci environment we only want to be publishing our
                    * containers from artifacts
                    */
                    stage('Publish') {
                        infra.withDockerCredentials {
                            sh 'make publish'
                        }
                    }
                }

                // Let's always clean up the docker images at the very end
                sh(script: 'docker system prune --force --all', returnStatus: true)
            }
        }
    }

    parallel builds
}


void nodeWithTimeout(String label, def body) {
    node(label) {
        timeout(time: 60, unit: 'MINUTES') {
            body.call()
        }
    }
}
