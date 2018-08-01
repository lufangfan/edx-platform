def runPythonTests() {
    ansiColor('gnome-terminal') {
        sshagent(credentials: ['jenkins-worker', 'jenkins-worker-pem'], ignoreMissing: true) {
            checkout changelog: false, poll: false, scm: [$class: 'GitSCM', branches: [[name: '${sha1}']],
                doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                userRemoteConfigs: [[credentialsId: 'jenkins-worker',
                refspec: '+refs/heads/*:refs/remotes/origin/* +refs/pull/*:refs/remotes/origin/pr/*',
                url: 'git@github.com:edx/edx-platform.git']]]
            sh 'bash scripts/all-tests.sh'
            // dir('stdout') {
            //     writeFile file: "${TEST_SUITE}-stdout.log", text: console_output
            // }
            stash includes: 'reports/**/*coverage*', name: "${TEST_SUITE}-reports"
        }
    }
}

def pythonTestCleanup() {
    archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/**/*,test_root/log/**/*.log,**/nosetests.xml,stdout/*.log,*.log'
    junit '**/nosetests.xml'
    sh '''source $HOME/edx-venv/bin/activate
    bash scripts/xdist/terminate_xdist_nodes.sh'''
}

pipeline {
    agent { label "coverage-worker" }
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
        stage('Run Tests') {
            parallel {
                stage("lms-unit") {
                    agent { label "jenkins-worker" }
                    environment {
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
                stage("cms-unit") {
                    agent { label "jenkins-worker" }
                    environment {
                        TEST_SUITE = "cms-unit"
                        XDIST_FILE_NAME_PREFIX = "${TEST_SUITE}"
                        XDIST_NUM_TASKS = 5
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
                stage("commonlib-unit") {
                    agent { label "jenkins-worker" }
                    environment {
                        TEST_SUITE = "commonlib-unit"
                        XDIST_FILE_NAME_PREFIX = "${TEST_SUITE}"
                        XDIST_NUM_TASKS = 3
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
            }
        }
        stage('Run coverage') {
            environment {
                CODE_COV_TOKEN = credentials('CODE_COV_TOKEN')
                TARGET_BRANCH = "origin/master"
                CI_BRANCH = "${ghprbSourceBranch}"
                SUBSET_JOB = "null"
            }
            steps {
                ansiColor('gnome-terminal') {
                    sshagent(credentials: ['jenkins-worker'], ignoreMissing: true) {
                        checkout changelog: false, poll: false, scm: [$class: 'GitSCM', branches: [[name: '${sha1}']],
                            doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                            userRemoteConfigs: [[credentialsId: 'jenkins-worker',
                            refspec: '+refs/heads/*:refs/remotes/origin/* +refs/pull/*:refs/remotes/origin/pr/*',
                            url: 'git@github.com:edx/edx-platform.git']]]
                        unstash 'lms-unit-reports'
                        unstash 'cms-unit-reports'
                        unstash 'commonlib-unit-reports'
                        sh "./scripts/jenkins-report.sh"
                    }
                }
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true,
                        reportDir: 'reports', reportFiles: 'diff_coverage_combined.html',
                        reportName: 'Diff Coverage Report', reportTitles: ''])
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true,
                        reportDir: 'reports/cover', reportFiles: 'index.html',
                        reportName: 'Coverage.py Report', reportTitles: ''])
                }
            }
        }
    }
}
