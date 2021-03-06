properties([
    buildDiscarder(logRotator(
        artifactDaysToKeepStr: '14',
        artifactNumToKeepStr: '30',
        daysToKeepStr: '14',
        numToKeepStr: '30',
    )),
    disableConcurrentBuilds(),
    pipelineTriggers([]),
    parameters([
        booleanParam(name: 'RUN_E2E_TESTS', defaultValue: true, description: 'If true, run e2e tests.'),
        booleanParam(name: 'SHORT_TESTS', defaultValue: false, description: 'If true, run tests with -test.short=true for running a subset of tests'),
        booleanParam(name: 'SKIP_DOCKER_STAGES', defaultValue: false, description: 'If true, skips docker build, tag and push'),
        booleanParam(name: 'SKIP_NAMESPACE_CLEANUP', defaultValue: false, description: 'If true, skips deleting the Kubernetes namespace at the end of the job'),
    ])
])

def isPullRequest = env.BRANCH_NAME.startsWith("PR-")
def isMasterBranch = env.BRANCH_NAME == "master"

def shortTests = params.SHORT_TESTS || (isPullRequest && pullRequest.labels.contains("run-short-tests"))
def skipNamespaceCleanup = params.SKIP_NAMESPACE_CLEANUP || (isPullRequest && pullRequest.labels.contains("skip-namespace-cleanup"))

def skipBuildLabel = (isPullRequest && pullRequest.labels.contains("skip-build"))
def skipBuild = params.SKIP_DOCKER_STAGES || skipBuildLabel

def runE2ETests = isMasterBranch || params.RUN_E2E_TESTS || (isPullRequest && pullRequest.labels.contains("run-e2e-tests"))

// skip-build GH label takes precedence over run-e2e-tests and params for running tests.
if (skipBuild && runE2ETests && skipBuildLabel) {
    runE2ETests = false
}

def branchTag = env.BRANCH_NAME.toLowerCase()
def deployTag = "${branchTag}-${currentBuild.number}"
def meteringNamespacePrefix = "metering-ci-${branchTag}"
def meteringE2ENamespace = "${meteringNamespacePrefix}-e2e"
def meteringIntegrationNamespace = "${meteringNamespacePrefix}-integration"

def instanceCap = isMasterBranch ? 1 : 5
def podLabel = "kube-metering-build-${isMasterBranch ? 'master' : 'pr'}"

def awsBillingBucket = "team-chargeback"
def awsBillingBucketPrefix = "cost-usage-report/team-chargeback-chancez/"
def enableAWSBilling = true

echo "Params:\n${params}"

podTemplate(
    cloud: 'kubernetes',
    containers: [
        containerTemplate(
            alwaysPullImage: false,
            envVars: [],
            command: 'dockerd-entrypoint.sh',
            args: '--storage-driver=overlay',
            image: 'docker:dind',
            name: 'docker',
            privileged: true,
            ttyEnabled: true,
        ),
    ],
    volumes: [
        emptyDirVolume(
            mountPath: '/var/lib/docker',
            memory: false,
        ),
    ],
    idleMinutes: 15,
    instanceCap: 5,
    label: podLabel,
    name: podLabel,
) {
    node (podLabel) {
    timestamps {
        def gopath = "${env.WORKSPACE}/go"
        def meteringSourceDir = "${gopath}/src/github.com/operator-framework/operator-metering"
        def testOutputDir = "test_output"
        def testOutputDirAbsolutePath = "${env.WORKSPACE}/${testOutputDir}"

        def e2eTestLogFile = 'e2e-tests.log'
        def e2eDeployLogFile = 'e2e-deploy.log'
        def e2eTestTapFile = 'e2e-tests.tap'
        def e2eDeployPodLogsFile = 'e2e-deploy-pod-logs.log'
        def e2eFinalPodLogsFile = 'e2e-final-pod-logs.log'

        def integrationTestLogFile = 'integration-tests.log'
        def integrationDeployLogFile = 'integration-deploy.log'
        def integrationTestTapFile = 'integration-tests.tap'
        def integrationDeployPodLogsFile = 'integration-deploy-pod-logs.log'
        def integrationFinalPodLogsFile = 'integration-final-pod-logs.log'

        def dockerBuildArgs = ''
        if (isMasterBranch) {
            dockerBuildArgs = '--no-cache'
        }

        try {
            container('docker'){

                stage('checkout') {
                    sh '''
                    apk update
                    apk add git bash jq zip python py-pip
                    pip install pyyaml
                    '''

                    checkout([
                        $class: 'GitSCM',
                        branches: scm.branches,
                        extensions: scm.extensions + [[$class: 'RelativeTargetDirectory', relativeTargetDir: meteringSourceDir]],
                        userRemoteConfigs: scm.userRemoteConfigs
                    ])
                }
            }

            withCredentials([
                [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'kube-chargeback-s3', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'],
                usernamePassword(credentialsId: 'quay-coreos-jenkins-push', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME'),
            ]) {
                withEnv([
                    "JENKINS_WORKSPACE=${env.WORKSPACE}",
                    "GOPATH=${gopath}",
                    "USE_LATEST_TAG=${isMasterBranch}",
                    "USE_RELEASE_TAG=${isMasterBranch}",
                    "PUSH_RELEASE_TAG=${isMasterBranch}",
                    "BRANCH_TAG=${branchTag}",
                    "DEPLOY_TAG=${deployTag}",
                    "BRANCH_TAG_CACHE=${isMasterBranch}",
                    "DOCKER_BUILD_ARGS=${dockerBuildArgs}",
                    "METERING_E2E_NAMESPACE=${meteringE2ENamespace}",
                    "METERING_INTEGRATION_NAMESPACE=${meteringIntegrationNamespace}",
                    "METERING_SHORT_TESTS=${shortTests}",
                    "AWS_BILLING_BUCKET=${awsBillingBucket}",
                    "AWS_BILLING_BUCKET_PREFIX=${awsBillingBucketPrefix}",
                    "AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}",
                    "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}",
                    "CLEANUP_METERING=${!skipNamespaceCleanup}",
                ]){
                    container('docker'){
                        echo "Authenticating to docker registry"
                        sh 'docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD quay.io'

                        stage('install dependencies') {
                            // Build & install thrift
                            sh '''#!/bin/bash
                            set -e
                            apk add make go libc-dev curl
                            export HELM_VERSION=2.8.0
                            curl \
                                --silent \
                                --show-error \
                                --location \
                                "https://storage.googleapis.com/kubernetes-helm/helm-v${HELM_VERSION}-linux-amd64.tar.gz" \
                                | tar xz --strip-components=1 -C /usr/local/bin linux-amd64/helm \
                                && chmod +x /usr/local/bin/helm
                            helm init --client-only --skip-refresh
                            helm repo remove stable || true

                            export KUBERNETES_VERSION=1.8.3
                            curl \
                                --silent \
                                --show-error \
                                --location \
                                "https://storage.googleapis.com/kubernetes-release/release/v${KUBERNETES_VERSION}/bin/linux/amd64/kubectl" \
                                -o /usr/local/bin/kubectl \
                                 && chmod +x /usr/local/bin/kubectl
                            '''
                        }

                        dir(meteringSourceDir) {
                            stage('test') {
                                sh '''#!/bin/bash
                                set -e
                                set -o pipefail
                                make ci-validate
                                make test
                                '''
                            }

                            stage('build') {
                                if (skipBuild) {
                                    echo "Skipping docker build"
                                } else {
                                    ansiColor('xterm') {
                                        sh '''#!/bin/bash -ex
                                        make docker-build-all -j 2 \
                                            BRANCH_TAG_CACHE=${BRANCH_TAG_CACHE} \
                                            USE_LATEST_TAG=${USE_LATEST_TAG} \
                                            BRANCH_TAG=${BRANCH_TAG} \
                                            DEPLOY_TAG=${DEPLOY_TAG}
                                        '''
                                    }
                                }
                            }

                            stage('tag') {
                                if (skipBuild) {
                                    echo "Skipping docker tag"
                                } else {
                                    ansiColor('xterm') {
                                        sh '''#!/bin/bash -ex
                                        make docker-tag-all -j 2
                                        '''
                                    }
                                }
                            }

                            stage('push') {
                                if (skipBuild) {
                                    echo "Skipping docker push"
                                } else {
                                    sh '''#!/bin/bash -ex
                                    make docker-push-all -j 2 \
                                        USE_LATEST_TAG=${USE_LATEST_TAG} \
                                        PUSH_RELEASE_TAG=${PUSH_RELEASE_TAG} \
                                        BRANCH_TAG=${BRANCH_TAG} \
                                        DEPLOY_TAG=${DEPLOY_TAG}
                                    '''
                                }
                            }
                        }
                    }

                    stage('integration/e2e tests') {
                        withCredentials([
                            [$class: 'FileBinding', credentialsId: 'chargeback-ci-kubeconfig', variable: 'TECTONIC_KUBECONFIG'],
                            [$class: 'FileBinding', credentialsId: 'openshift-chargeback-ci-kubeconfig', variable: 'OPENSHIFT_KUBECONFIG'],
                            [$class: 'FileBinding', credentialsId: 'gke-metering-ci-kubeconfig', variable: 'GKE_KUBECONFIG'],
                        ]) {
                            parallel "generic-e2e": {
                                if (runE2ETests) {
                                    echo "Running metering e2e tests"
                                    def myTestDir = "${testOutputDir}/generic_e2e"
                                    def myTestDirAbs = "${testOutputDirAbsolutePath}/generic_e2e"
                                    def testReportResultsDir = "${myTestDirAbs}/test_report_results"
                                    container('docker') {
                                        sh "mkdir -p ${testReportResultsDir}"
                                    }
                                    e2eRunner(meteringSourceDir, [
                                        "METERING_NAMESPACE=${METERING_E2E_NAMESPACE}",
                                        "KUBECONFIG=${GKE_KUBECONFIG}",
                                        "TEST_OUTPUT_DIR=${myTestDirAbs}",
                                        "TEST_LOG_FILE=${e2eTestLogFile}",
                                        "TEST_RESULT_REPORT_OUTPUT_DIRECTORY=${testReportResultsDir}",
                                        "DEPLOY_LOG_FILE=${e2eDeployLogFile}",
                                        "DEPLOY_POD_LOGS_LOG_FILE=${e2eDeployPodLogsFile}",
                                        "FINAL_POD_LOGS_LOG_FILE=${e2eFinalPodLogsFile}",
                                        "DEPLOY_PLATFORM=generic",
                                        "TEST_TAP_FILE=${e2eTestTapFile}",
                                        "ENTRYPOINT=hack/e2e-ci.sh",
                                    ], skipNamespaceCleanup)
                                    step([$class: "TapPublisher", testResults: "${myTestDir}/${e2eTestTapFile}", failIfNoResults: false, planRequired: false])
                                } else {
                                    echo "Non-master branch, skipping metering e2e tests"
                                }
                            }, "generic-integration": {
                                if (runE2ETests) {
                                    echo "Running metering integration tests"
                                    def myTestDir = "${testOutputDir}/generic_integration"
                                    def myTestDirAbs = "${testOutputDirAbsolutePath}/generic_integration"
                                    def testReportResultsDir = "${myTestDirAbs}/test_report_results"
                                    container('docker') {
                                        sh "mkdir -p ${testReportResultsDir}"
                                    }
                                    e2eRunner(meteringSourceDir, [
                                        "METERING_NAMESPACE=${METERING_INTEGRATION_NAMESPACE}",
                                        "KUBECONFIG=${GKE_KUBECONFIG}",
                                        "TEST_OUTPUT_DIR=${myTestDirAbs}",
                                        "TEST_RESULT_REPORT_OUTPUT_DIRECTORY=${testReportResultsDir}",
                                        "TEST_LOG_FILE=${integrationTestLogFile}",
                                        "DEPLOY_LOG_FILE=${integrationDeployLogFile}",
                                        "DEPLOY_POD_LOGS_LOG_FILE=${integrationDeployPodLogsFile}",
                                        "FINAL_POD_LOGS_LOG_FILE=${integrationFinalPodLogsFile}",
                                        "DEPLOY_PLATFORM=generic",
                                        "TEST_TAP_FILE=${integrationTestTapFile}",
                                        "ENTRYPOINT=hack/integration-ci.sh",
                                    ], skipNamespaceCleanup)
                                    step([$class: "TapPublisher", testResults: "${myTestDir}/${integrationTestTapFile}", failIfNoResults: false, planRequired: false])
                                } else {
                                    echo "Non-master branch, skipping metering integration tests"
                                }
                            }, "openshift-e2e": {
                                if (runE2ETests) {
                                    echo "Running metering e2e tests"
                                    def myTestDir = "${testOutputDir}/openshift_e2e"
                                    def myTestDirAbs = "${testOutputDirAbsolutePath}/openshift_e2e"
                                    def testReportResultsDir = "${myTestDirAbs}/test_report_results"
                                    container('docker') {
                                        sh "mkdir -p ${testReportResultsDir}"
                                    }
                                    e2eRunner(meteringSourceDir, [
                                        "METERING_NAMESPACE=${METERING_E2E_NAMESPACE}",
                                        "KUBECONFIG=${OPENSHIFT_KUBECONFIG}",
                                        "TEST_OUTPUT_DIR=${myTestDirAbs}",
                                        "TEST_LOG_FILE=${e2eTestLogFile}",
                                        "TEST_RESULT_REPORT_OUTPUT_DIRECTORY=${testReportResultsDir}",
                                        "DEPLOY_LOG_FILE=${e2eDeployLogFile}",
                                        "DEPLOY_POD_LOGS_LOG_FILE=${e2eDeployPodLogsFile}",
                                        "ENABLE_AWS_BILLING=${enableAWSBilling}",
                                        "ENABLE_AWS_BILLING_TEST=${enableAWSBilling}",
                                        "DEPLOY_PLATFORM=openshift",
                                        "TEST_TAP_FILE=${e2eTestTapFile}",
                                        "ENTRYPOINT=hack/e2e-ci.sh",
                                    ], skipNamespaceCleanup)
                                    step([$class: "TapPublisher", testResults: "${myTestDir}/${e2eTestTapFile}", failIfNoResults: false, planRequired: false])
                                } else {
                                    echo "Non-master branch, skipping metering e2e tests"
                                }
                            }, "openshift-integration": {
                                if (runE2ETests) {
                                    echo "Running metering integration tests"
                                    def myTestDir = "${testOutputDir}/openshift_integration"
                                    def myTestDirAbs = "${testOutputDirAbsolutePath}/openshift_integration"
                                    def testReportResultsDir = "${myTestDirAbs}/test_report_results"
                                    container('docker') {
                                        sh "mkdir -p ${testReportResultsDir}"
                                    }
                                    e2eRunner(meteringSourceDir, [
                                        "METERING_NAMESPACE=${METERING_INTEGRATION_NAMESPACE}",
                                        "KUBECONFIG=${OPENSHIFT_KUBECONFIG}",
                                        "TEST_OUTPUT_DIR=${myTestDirAbs}",
                                        "TEST_RESULT_REPORT_OUTPUT_DIRECTORY=${testReportResultsDir}",
                                        "TEST_LOG_FILE=${integrationTestLogFile}",
                                        "DEPLOY_LOG_FILE=${integrationDeployLogFile}",
                                        "DEPLOY_POD_LOGS_LOG_FILE=${integrationDeployPodLogsFile}",
                                        "DEPLOY_PLATFORM=openshift",
                                        "ENABLE_AWS_BILLING=${enableAWSBilling}",
                                        "ENABLE_AWS_BILLING_TEST=${enableAWSBilling}",
                                        "TEST_TAP_FILE=${integrationTestTapFile}",
                                        "ENTRYPOINT=hack/integration-ci.sh",
                                    ], skipNamespaceCleanup)
                                    step([$class: "TapPublisher", testResults: "${myTestDir}/${integrationTestTapFile}", failIfNoResults: false, planRequired: false])
                                } else {
                                    echo "Non-master branch, skipping metering integration tests"
                                }
                            }, "tectonic-e2e": {
                                if (runE2ETests) {
                                    echo "Running metering e2e tests"
                                    def myTestDir = "${testOutputDir}/tectonic_e2e"
                                    def myTestDirAbs = "${testOutputDirAbsolutePath}/tectonic_e2e"
                                    def testReportResultsDir = "${myTestDirAbs}/test_report_results"
                                    container('docker') {
                                        sh "mkdir -p ${testReportResultsDir}"
                                    }
                                    e2eRunner(meteringSourceDir, [
                                        "METERING_NAMESPACE=${METERING_E2E_NAMESPACE}",
                                        "KUBECONFIG=${TECTONIC_KUBECONFIG}",
                                        "TEST_OUTPUT_DIR=${myTestDirAbs}",
                                        "TEST_LOG_FILE=${e2eTestLogFile}",
                                        "TEST_RESULT_REPORT_OUTPUT_DIRECTORY=${testReportResultsDir}",
                                        "DEPLOY_LOG_FILE=${e2eDeployLogFile}",
                                        "DEPLOY_POD_LOGS_LOG_FILE=${e2eDeployPodLogsFile}",
                                        "FINAL_POD_LOGS_LOG_FILE=${e2eFinalPodLogsFile}",
                                        "ENABLE_AWS_BILLING=${enableAWSBilling}",
                                        "ENABLE_AWS_BILLING_TEST=${enableAWSBilling}",
                                        "DEPLOY_PLATFORM=tectonic",
                                        "TEST_TAP_FILE=${e2eTestTapFile}",
                                        "ENTRYPOINT=hack/e2e-ci.sh",
                                    ], skipNamespaceCleanup)
                                    step([$class: "TapPublisher", testResults: "${myTestDir}/${e2eTestTapFile}", failIfNoResults: false, planRequired: false])
                                } else {
                                    echo "Non-master branch, skipping metering e2e tests"
                                }
                            }, "tectonic-integration": {
                                if (runE2ETests) {
                                    echo "Running metering integration tests"
                                    def myTestDir = "${testOutputDir}/tectonic_integration"
                                    def myTestDirAbs = "${testOutputDirAbsolutePath}/tectonic_integration"
                                    def testReportResultsDir = "${myTestDirAbs}/test_report_results"
                                    container('docker') {
                                        sh "mkdir -p ${testReportResultsDir}"
                                    }
                                    e2eRunner(meteringSourceDir, [
                                        "METERING_NAMESPACE=${METERING_INTEGRATION_NAMESPACE}",
                                        "KUBECONFIG=${TECTONIC_KUBECONFIG}",
                                        "TEST_OUTPUT_DIR=${myTestDirAbs}",
                                        "TEST_RESULT_REPORT_OUTPUT_DIRECTORY=${testReportResultsDir}",
                                        "TEST_LOG_FILE=${integrationTestLogFile}",
                                        "DEPLOY_LOG_FILE=${integrationDeployLogFile}",
                                        "DEPLOY_POD_LOGS_LOG_FILE=${integrationDeployPodLogsFile}",
                                        "FINAL_POD_LOGS_LOG_FILE=${integrationFinalPodLogsFile}",
                                        "DEPLOY_PLATFORM=tectonic",
                                        "ENABLE_AWS_BILLING=${enableAWSBilling}",
                                        "ENABLE_AWS_BILLING_TEST=${enableAWSBilling}",
                                        "TEST_TAP_FILE=${integrationTestTapFile}",
                                        "ENTRYPOINT=hack/integration-ci.sh",
                                    ], skipNamespaceCleanup)
                                    step([$class: "TapPublisher", testResults: "${myTestDir}/${integrationTestTapFile}", failIfNoResults: false, planRequired: false])
                                } else {
                                    echo "Non-master branch, skipping metering integration tests"
                                }
                            }, failFast: false
                        }
                    }
                }
            }
        } catch (e) {
            // If there was an exception thrown, the build failed
            echo "Build failed"
            currentBuild.result = "FAILED"
            throw e
        } finally {
            if (runE2ETests) {
                archiveArtifacts artifacts: "${testOutputDir}/**", onlyIfSuccessful: false
                container('docker') {
                    sh '((docker ps -aq | xargs docker kill) || true) > /dev/null 2>&1'
                }
            }
            cleanWs notFailBuild: true
        }
    }
} // timestamps end
} // podTemplate end

def e2eRunner(meteringSourceDir, envVars, skipNamespaceCleanup) {
    withEnv(envVars) {
        container('docker'){
            dir(meteringSourceDir) {
                sh 'kubectl config current-context'
                sh 'kubectl config get-contexts'
                try {
                    ansiColor('xterm') {
                        timeout(20) {
                            sh '''#!/bin/bash -ex
                            mkdir -p ${TEST_OUTPUT_DIR}
                            touch ${TEST_OUTPUT_DIR}/${DEPLOY_LOG_FILE}
                            touch ${TEST_OUTPUT_DIR}/${TEST_LOG_FILE}
                            tail -f ${TEST_OUTPUT_DIR}/${DEPLOY_LOG_FILE} &
                            tail -f ${TEST_OUTPUT_DIR}/${TEST_LOG_FILE} &
                            docker run \
                            -i --rm \
                            --env-file <(env | grep -E 'INSTALL_METHOD|TEST|LOG|DEPLOY|KUBECONFIG|AWS|CHARGEBACK|METERING|PULL_SECRET') \
                            -v "${JENKINS_WORKSPACE}:${JENKINS_WORKSPACE}" \
                            -v "${KUBECONFIG}:${KUBECONFIG}" \
                            -v "${TEST_OUTPUT_DIR}:/out" \
                            quay.io/coreos/chargeback-integration-tests:${DEPLOY_TAG} \
                            ${ENTRYPOINT}
                            '''
                        }
                    }
                } finally {
                    if (!skipNamespaceCleanup) {
                        sh '''#!/bin/bash -e
                        ./hack/delete-ns.sh
                        '''
                    }
                }
            }
        }
    }
}
