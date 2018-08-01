def runPythonTests() {
    ansiColor('gnome-terminal') {
        sshagent(credentials: ['jenkins-worker', 'jenkins-worker-pem'], ignoreMissing: true) {
            console_output = sh returnStdout: true, script: '''source $HOME/edx-venv/bin/activate
            bash scripts/generic-ci-tests.sh'''
            dir('stdout') {
                writeFile file: "${TEST_SUITE}-stdout.log", text: console_output
            }
            stash includes: 'reports/**/*coverage*', name: "${TEST_SUITE}-reports"
        }
    }
}

def pythonTestCleanup() {
    sh '''source $HOME/edx-venv/bin/activate
    bash scripts/xdist/terminate_xdist_nodes.sh'''
    archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**/*,test_root/log/**/*.log,**/nosetests.xml,stdout/*.log,*.log'
    junit '**/nosetests.xml'
}

pipeline {
    agent { label "jenkins-worker" }
    options {
        timestamps()
        timeout(60)
    }
    environment {
        XDIST_CONTAINER_SUBNET = credentials('XDIST_CONTAINER_SUBNET')
        XDIST_CONTAINER_SECURITY_GROUP = credentials('XDIST_CONTAINER_SECURITY_GROUP')
        XDIST_CONTAINER_TASK_NAME = "jenkins-worker-task"
        XDIST_GIT_BRANCH = "${ghprbActualCommit}"
    }
    stages {
        stage("Test setup"){
            steps {
                sshagent(credentials: ['jenkins-worker'], ignoreMissing: true) {
                    checkout changelog: false, poll: false, scm: [$class: 'GitSCM', branches: [[name: '${sha1}']],
                        doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                        userRemoteConfigs: [[credentialsId: 'jenkins-worker',
                        refspec: '+refs/heads/*:refs/remotes/origin/* +refs/pull/*:refs/remotes/origin/pr/*',
                        url: 'git@github.com:edx/edx-platform.git']]]
                    sh '''source scripts/jenkins-common.sh
                    paver install_python_prereqs'''
                }
            }
        }
        stage('Run Tests') {
            parallel {
                stage("lms-unit") {
                    environment {
                        // NO_PREREQ_INSTALL="True"
                        TEST_SUITE = "lms-unit"
                        XDIST_FILE_NAME_PREFIX = "${TEST_SUITE}"
                        XDIST_NUM_TASKS = 10
                    }
                    steps {
                        script {
                            runPythonTests()
                        }
                    }
                    post {
                        always {
                            script {
                                pythonTestCleanup()
                            }
                        }
                    }
                }
                // stage("cms-unit") {
                //     environment {
                //         NO_PREREQ_INSTALL="True"
                //         TEST_SUITE = "cms-unit"
                //         XDIST_FILE_NAME_PREFIX = "${TEST_SUITE}"
                //         XDIST_NUM_TASKS = 2
                //     }
                //     steps {
                //         script {
                //             runPythonTests()
                //         }
                //     }
                //     post {
                //         always {
                //             script {
                //                 pythonTestCleanup()
                //             }
                //         }
                //     }
                // }
                // stage("commonlib-unit") {
                //     environment {
                //         NO_PREREQ_INSTALL="True"
                //         TEST_SUITE = "commonlib-unit"
                //         XDIST_FILE_NAME_PREFIX = "${TEST_SUITE}"
                //         XDIST_NUM_TASKS = 3
                //     }
                //     steps {
                //         script {
                //             runPythonTests()
                //         }
                //     }
                //     post {
                //         always {
                //             script {
                //                 pythonTestCleanup()
                //             }
                //         }
                //     }
                // }
            }
        }
    }
}
