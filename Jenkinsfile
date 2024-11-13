properties([
    parameters([
        string(name: 'externalAppId', defaultValue: '', description: 'External App ID in Velocity'),
        choice(name: 'brandName', choices: ['hcl', 'ibm'], description: 'The brand of the deployed VM. Either hcl or ibm.')
    ])
])
node {
    ansiColor('xterm') {
      def commitId

        stage('Checkout') {
            def scmVars = checkout scm
            commitId = scmVars.GIT_COMMIT
        }
        stage('Prep') {
            /* pull down source code */
            def scmVars = checkout scm
            currentCommit = scmVars.GIT_COMMIT

            /* update build info */

            accelerateVersion = sh(
                script: 'cat orchestration/docker-compose/.env | grep ACCELERATE_VERSION= | sed -n -e "s/^.*ACCELERATE_VERSION=//p"',
                returnStdout: true
            ).trim()
            velocityVersion = sh(
                script: 'cat orchestration/docker-compose/.env | grep VELOCITY_VERSION= | sed -n -e "s/^.*VELOCITY_VERSION=//p"',
                returnStdout: true
            ).trim()
            productVersion = accelerateVersion
            changeId = env.CHANGE_ID
            if (isPr && gitHub.doesPrLabelExist(prNumber: changeId, labelName: 'CI: IBM VM')) {
                brandName = 'ibm'
            }
            currentBuild.displayName = "${productVersion}.${env.BUILD_ID}"

            def skipBuildLabel = 'CI: Skip Build'
            if (isPr && gitHub.doesPrLabelExist(prNumber: changeId, labelName: skipBuildLabel)) {
                error("Skipping build due to existence of '${skipBuildLabel}' label on GitHub Pull Request")
            }

            commitMessage = sh(
                script: 'git log -1 --oneline --pretty=%B',
                returnStdout: true
            ).trim()
            userEmail = sh(
                script: 'git --no-pager show -s --format=\'%ae\'',
                returnStdout: true
            ).trim()

            if (isPr) {
                sinceCommitHash = gitHub.getPrBaseRef(prNumber: changeId)
            } else {
                def successfulCommit = getLastSuccessfulCommit(gitBranch)
                if (successfulCommit) {
                    sinceCommitHash = successfulCommit
                } else {
                    println 'Could not determine last successful commit built in Jenkins, using initial commit'
                    sinceCommitHash = initialCommit
                }
            }

            lastSuccessfulCommitHash = sinceCommitHash
            println "lastSuccessfulCommitHash: '${lastSuccessfulCommitHash}'"

            if (params.BuildAll) {
                sinceCommitHash = initialCommit
            }
            println "sinceCommitHash: '${sinceCommitHash}'"

            sourceVersion
            if (isPr) {
                sourceVersion = "pr/${changeId}"
            } else {
                sourceVersion = gitBranch
            }
            sh 'git config --global user.name "Velocity Build"'

            artifactPathOverride = currentBuild.displayName
            if (isPr) {
                artifactPathOverride = gitBranch
            }

            sh 'nohup /usr/local/bin/dockerd --host=unix:///var/run/docker.sock --host=tcp://127.0.0.1:2375 --storage-driver=overlay2&'
            sh 'timeout 15 sh -c "until docker info; do echo .; sleep 1; done"'
            sh 'apt-get install -y xmlstarlet'

            withAWS(credentials:'aws-velocity-ecr') {
                sh "${ecrLogin()}"
            }
        }

        stage('Install') {
            sh 'yarn install --immutable --immutable-cache'
        }

        withEnv([
            'METEOR_ALLOW_SUPERUSER=true',
            'ACCESS_KEY=U2FsdGVkX19aIg1spAFIniaQ79UEWiHPFUXzrJiK5OGeXVqD+i6VM1QXRmJgFtKT9aY35F3owfcZhywzXmaWVw==',
            "BUILD_NUMBER=${currentBuild.displayName}",
            "LAST_SUCCESSFUL_COMMIT=${lastSuccessfulCommitHash}",
            "BRANCH_NAME=${sourceVersion}",
            "SINCE_COMMIT=${sinceCommitHash}",
            "JENKINS_BUILD_URL=${env.BUILD_URL}",
            "PRODUCT_VERSION=${productVersion}",
            "IBM_VERSION=${velocityVersion}.${env.BUILD_ID}",
            "VELOCITY_BUILD=${env.BUILD_ID}",
            "SKIP_PUBLISH=${params.SkipPublish}",
            "MONGO_VERSION_OVERRIDE=${params.MongoVersionOverride}",
            "SKIP_SONAR=${isReleaseBuild || params.SkipPublish}",
            "IS_NOT_PR=${!isPr}",
            "METEOR_TAG=${gitBranch}-${currentBuild.displayName}",
            'DOCKER_REGISTRY_PREFIX=082163301176.dkr.ecr.us-east-1.amazonaws.com/uc-velocity/'
        ]) {
            try {
                timeout(time: 90, unit: 'MINUTES') {
                    parallel(
                        'local': {
                            stage('Run Local Build') {
                                runDownstreamTests('Velocity SE/Local Build', 'Local Build', !isPr && !params.SkipPublish, !params.SkipPublish, isPr, [
                                    string(name: 'branchName', value: gitBranch),
                                    string(name: 'commitId', value: currentCommit),
                                    string(name: 'productVersion', value: currentBuild.displayName),
                                    booleanParam(name: 'SkipPublish', value: params.SkipPublish),
                                    string(name: 'metricsBuildUrl', value: env.BUILD_URL),
                                    string(name: 'metricsBuildId', value: currentBuild.displayName),
                                ], [
                                // do not save tests here as do not want unit/func failed tests to fail the build
                                // since those are handled by separate GitHub badge/check now and gated on in our Pipeline
                                ], 'local-func-unit-test-results', false)
                            }
                        },
                        'docker': {
                            parallel(
                                'node': {
                                    stage('Docker Base') {
                                        sh 'yarn build-base-images'
                                    }
                                    stage('Node Images') {
                                        withCredentials([string(credentialsId:'Anthill_HCL_Github_Access_Token', variable:'GITHUB_ACCESS_TOKEN')]) {
                                            sh 'yarn build-images-node'
                                        }
                                    }
                                },
                                'meteor': {
                                    stage('Build Meteor Images') {
                                        runDownstreamTests('Velocity SE/Meteor Build', 'Meteor Build',  true, !params.SkipPublish, isPr, [
                                            string(name: 'branchName', value: gitBranch),
                                            string(name: 'commitId', value: currentCommit),
                                            string(name: 'productVersion', value: currentBuild.displayName)
                                        ], [], '', true)
                                        sh "docker pull ${DOCKER_REGISTRY_PREFIX}cr-ui-temp:${METEOR_TAG}"
                                        sh "docker tag ${DOCKER_REGISTRY_PREFIX}cr-ui-temp:${METEOR_TAG} ${DOCKER_REGISTRY_PREFIX}continuous-release-ui:latest"
                                        sh "docker tag ${DOCKER_REGISTRY_PREFIX}cr-ui-temp:${METEOR_TAG} ${DOCKER_REGISTRY_PREFIX}continuous-release-ui:${BUILD_NUMBER}"
                                    }
                                }
                            )
                            stage('Tag Images') {
                                sh 'yarn tag-images'
                            }
                            stage('Installer Prep') {
                                dir('packages/helm-parser') {
                                    sh 'yarn build'
                                }
                                dir('orchestration/installer') {
                                    sh 'yarn install --immutable --immutable-cache'
                                    sh 'yarn package'
                                }
                            }
                            stage('Cloud Installer') {
                                dir('orchestration/installer') {
                                    sh 'yarn dist-hcl-offline'
                                    sh 'yarn dist-ibm-offline'
                                }
                            }
                            stage('Delete Previous Installers') {
                                withAWS(credentials:'aws-velocity-ecr') {
                                    def previousInstallersExist = s3DoesObjectExist(
                                        bucket: s3Bucket,
                                        path: "${artifactPathOverride}/velocity/${velCloudInstallLocation}"
                                    )
                                    if (previousInstallersExist) {
                                        s3Delete(
                                            bucket: s3Bucket,
                                            path: "${artifactPathOverride}/velocity/orchestration/dist/"
                                        )
                                    }
                                }
                            }
                            parallel(
                                'external': {
                                    stage('Upload Cloud Install') {
                                        withAWS(credentials:'aws-velocity-ecr') {
                                            // upload just the offline installers needed for e2e tests to the internal AWS S3 bucket (binaries)
                                            s3Upload(
                                                bucket: s3Bucket,
                                                path: "${artifactPathOverride}/velocity/${velCloudInstallLocation}",
                                                file: velCloudInstallLocation
                                            )
                                            s3Upload(
                                                bucket: s3Bucket,
                                                path: "${artifactPathOverride}/velocity/${accCloudInstallLocation}",
                                                file: accCloudInstallLocation
                                            )
                                        }
                                    }
                                    stage('Deploy') {
                                        def shouldKickOffDeploy = isPr && !gitHub.doesPrLabelExist(prNumber: changeId, labelName: 'CI: Skip Deployment')
                                        if (shouldKickOffDeploy) {
                                            println("Deploying ${brandName} server on pull request VM.")
                                            build(
                                                job: 'Velocity SE/Automated PR VM Deployment',
                                                parameters: [
                                                    string(name: 'pullRequestNumber', value: changeId),
                                                    string(name: 'userEmail', value: userEmail),
                                                    string(name: 'brandName', value: brandName)
                                                ],
                                                wait: false,
                                                quietPeriod: 0
                                            )
                                        }
                                    }
                                    stage('Perf Test') {
                                        def shouldKickOffPerf = isPr && isBuildSucceeding() && !gitHub.doesPrLabelExist(prNumber: changeId, labelName: 'CI: Skip Perf')
                                        if (shouldKickOffPerf) {
                                            build(
                                                job: 'Velocity SE/Performance Test',
                                                parameters: [
                                                    string(name: 'branchName', value: gitBranch),
                                                    string(name: 'commitId', value: currentCommit),
                                                    string(name: 'productVersion', value: currentBuild.displayName),
                                                    string(name: 'branding', value: 'HCL'),
                                                    booleanParam(name: 'skipPublish', value: params.SkipPublish),
                                                    string(name: 'metricsBuildUrl', value: env.BUILD_URL),
                                                    string(name: 'metricsBuildId', value: currentBuild.displayName),
                                                ],
                                                wait: false,
                                                quietPeriod: 0
                                            )
                                        }
                                    }
                                    stage('E2E Tests') {
                                        def shouldKickOffE2e = !isPr || (isBuildSucceeding() && !gitHub.doesPrLabelExist(prNumber: changeId, labelName: 'CI: Skip E2E'))
                                        if (shouldKickOffE2e) {
                                            parallel(
                                                'ibm': {
                                                    runDownstreamTests(e2eManagerJobName, 'IBM E2E', false, false, isPr, [
                                                        string(name: 'branchName', value: gitBranch),
                                                        string(name: 'commitId', value: currentCommit),
                                                        string(name: 'productVersion', value: currentBuild.displayName),
                                                        string(name: 'branding', value: 'IBM'),
                                                        string(name: 'concurrentExecutorsOverride', value: params.concurrentExecutorsOverride),
                                                        string(name: 'sequentialExecutorsOverride', value: params.sequentialExecutorsOverride),
                                                        booleanParam(name: 'skipPublish', value: params.SkipPublish),
                                                        string(name: 'metricsBuildUrl', value: env.BUILD_URL),
                                                        string(name: 'metricsBuildId', value: currentBuild.displayName),
                                                    ], [
                                                        [ path: 'concurrent-ibm.xml', title:'IBM concurrent' ],
                                                        [ path: 'hybrid-ucd-jenkins-ibm.xml', title: 'IBM hybrid-ucd-jenkins' ],
                                                        [ path: 'jenkins-ibm.xml', title: 'IBM jenkins' ],
                                                        [ path: 'ucd-ibm.xml', title: 'IBM ucd' ],
                                                        [ path: 'sequential-ibm.xml', title: 'IBM sequential' ],
                                                        [ path: 'api-ibm.xml', title: 'IBM Api' ]
                                                    ], 'e2e-ibm-test-results', false, 'test-results-final/*.xml')
                                                },
                                                'hcl': {
                                                    runDownstreamTests(e2eManagerJobName, 'HCL E2E', false, false, isPr, [
                                                        string(name: 'branchName', value: gitBranch),
                                                        string(name: 'commitId', value: currentCommit),
                                                        string(name: 'productVersion', value: currentBuild.displayName),
                                                        string(name: 'branding', value: 'HCL'),
                                                        string(name: 'concurrentExecutorsOverride', value: params.concurrentExecutorsOverride),
                                                        string(name: 'sequentialExecutorsOverride', value: params.sequentialExecutorsOverride),
                                                        booleanParam(name: 'skipPublish', value: params.SkipPublish),
                                                        string(name: 'metricsBuildUrl', value: env.BUILD_URL),
                                                        string(name: 'metricsBuildId', value: currentBuild.displayName),
                                                    ], [
                                                        [ path: 'concurrent-hcl.xml', title:'HCL concurrent' ],
                                                        [ path: 'hybrid-ucd-jenkins-hcl.xml', title: 'HCL hybrid-ucd-jenkins' ],
                                                        [ path: 'jenkins-hcl.xml', title: 'HCL jenkins' ],
                                                        [ path: 'ucd-hcl.xml', title: 'HCL ucd' ],
                                                        [ path: 'sequential-hcl.xml', title: 'HCL sequential' ],
                                                        [ path: 'api-hcl.xml', title: 'HCL Api' ]
                                                    ], 'e2e-hcl-test-results', false, 'test-results-final/*.xml')
                                                }
                                            )
                                        }
                                    }
                                },
                                'internal': {
                                    stage('Other Installers') {
                                        dir('orchestration/installer') {
                                            sh 'yarn dist-ibm'
                                            sh 'yarn dist-hcl'
                                        }
                                    }
                                    parallel(
                                        'installer-tests': {
                                            stage('Installer Tests') {
                                                try {
                                                    dir('orchestration/installer') {
                                                        sh 'yarn test:ci'
                                                    }
                                                } catch (Exception e) {
                                                    println 'Exception caught in installer test block:'
                                                    println e
                                                } finally {
                                                    saveTestResults(!params.SkipPublish, isPr, true, [
                                                        [ path: 'orchestration/installer/test-results/unit/results.xml', title: 'Installer Unit' ],
                                                        [ path: 'orchestration/installer/test-results/func/results.xml', title: 'Installer Func' ]
                                                    ])
                                                }
                                            }
                                        },
                                        'upload-installers': {
                                            stage('Upload Installers') {
                                                if (isBuildSucceeding() || isPr) {
                                                    withAWS(credentials:'aws-velocity-ecr') {
                                                        s3Upload(
                                                            bucket: s3Bucket,
                                                            path: "${artifactPathOverride}/velocity/orchestration/dist",
                                                            workingDir: 'orchestration/dist',
                                                            includePathPattern: '*-linux,*-macos,*-win.exe'
                                                        )
                                                    }
                                                }
                                            }
                                            if (!params.SkipPublish) {
                                                stage('WhiteSource') {
                                                    build(
                                                        job: 'Velocity SE/whitesource/WhiteSource Scan',
                                                        parameters: [
                                                            string(name: 'buildNumber', value: isPr ? gitBranch : currentBuild.displayName)
                                                        ],
                                                        wait: false,
                                                        quietPeriod: 0
                                                    )
                                                }
                                            }
                                        }
                                    )
                                }
                            )
                        }
                    )
                    withEnv([
                        "BUILD_RESULT=${currentBuild.currentResult}",
                    ]) {
                        stage('Publish') {
                            withAWS(credentials:'aws-velocity-ecr') {
                                sh "${ecrLogin()}"
                                sh 'yarn publish-images:ci'
                            }
                        }
                        stage('External Sync') {
                            def shouldKickOffExternalSync = isBuildNotFailed() && (!isPr || !gitHub.doesPrLabelExist(prNumber: changeId, labelName: 'CI: Skip External Sync'))
                            if (shouldKickOffExternalSync) {
                                def truncatedCommitMessage = commitMessage.take(100) + (commitMessage.length() > 100 ?  '...' : '')
                                parallel(
                                    'cloudpak': {
                                        runDownstreamTests('Velocity SE/CloudPak Sync', 'CloudPak Sync', false, !params.SkipPublish, isPr, [
                                            string(name: 'buildNumber', value: currentBuild.displayName),
                                            string(name: 'velocityVersion', value: velocityVersion),
                                            string(name: 'branchName', value: gitBranch),
                                            string(name: 'commitMessage', value: truncatedCommitMessage),
                                            string(name: 'commitId', value: currentCommit),
                                            booleanParam(name: 'push', value: !isPr && !params.SkipPublish)
                                        ], [], '')
                                    },
                                    'sofy': {
                                        runDownstreamTests('Velocity SE/SoFy-Sync', 'SoFy Sync', false, !params.SkipPublish, isPr, [
                                            string(name: 'buildNumber', value: currentBuild.displayName),
                                            string(name: 'branchName', value: gitBranch),
                                            string(name: 'accelerateVersion', value: accelerateVersion),
                                            string(name: 'commitMessage', value: truncatedCommitMessage),
                                            string(name: 'commitId', value: currentCommit),
                                            string(name: 'reviewers', value: ''),
                                            booleanParam(name: 'useLatestOnBranch', value: false),
                                            booleanParam(name: 'publishSofy', value: false),
                                            booleanParam(name: 'pushImagesSofy', value: false),
                                            booleanParam(name: 'skipInternalPublish', value: params.SkipPublish),
                                            string(name: 'metricsBuildUrl', value: env.BUILD_URL),
                                            string(name: 'metricsBuildId', value: currentBuild.displayName),
                                        ], [], '')
                                    }
                                )
                            } else {
                                println 'Not syncing externally due to the following conditions:'
                                println "isBuildNotFailed(): ${isBuildNotFailed()}"
                            }
                        }
                        stage('Publish Packages') {
                            if (isMasterBuild && !params.SkipPublish && isBuildNotFailed()) {
                                build(
                                    job: 'Velocity SE/Publish Packages',
                                    wait: false,
                                    quietPeriod: 60 // seconds
                                )
                            }
                        }
                    }
                }
            } catch (Exception e) {
                println "Build failed: ${e}"
                currentBuild.result = hudson.model.Result.FAILURE.toString()
            } finally {
                try {
                    if (params.MongoVersionOverride != '' && !isBuildSucceeding()) {
                        stage('Notify Teams') {
                            office365ConnectorSend(
                                message: "Nightly Mongo test failed ${productVersion}. [Details](${env.RUN_DISPLAY_URL})",
                                webhookUrl: velocityTeamNotificationChannelWebhook
                            )
                        }
                    }

                    if (isPr && !isBuildSucceeding()) {
                        stage('Failure Email') {
                            println "Sending failure alert email to ${userEmail}..."
                            emailext(
                                to: userEmail,
                                body: "There was a problem with the build for: ${currentBuild.fullDisplayName}. \n\nCheck Jenkins for details: ${env.RUN_DISPLAY_URL}",
                                subject: "Jenkins Build Failed: ${currentBuild.fullDisplayName}"
                            )
                        }
                    }
                } catch (Exception ex) {
                    println "Build notifications failed: ${ex}"
                    currentBuild.result = hudson.model.Result.FAILURE.toString()
                }
            }
        }
    }
}

def isBuildSucceeding() {
    return hudson.model.Result.SUCCESS.toString() == currentBuild.currentResult
}

def isBuildNotFailed() {
    return isBuildSucceeding() || hudson.model.Result.UNSTABLE.toString() == currentBuild.currentResult
}

def jenkinsBuildResultToVelocityStatus(result) {
    if (
        result == hudson.model.Result.SUCCESS.toString() ||
        result == hudson.model.Result.FAILURE.toString() ||
        result == hudson.model.Result.UNSTABLE.toString()
    ) {
        return result.toLowerCase()
    }
    return 'failure'
}

@NonCPS
def isReleaseBranch(branch) {
    def isReleaseMatcher = branch =~ /\d+\.\d+\.\d+/
    return isReleaseMatcher.matches()
}

@NonCPS
def getLastSuccessfulBuild(branch) {
    def buildName = "Velocity SE/velocity-core/${branch}"
    def build = Jenkins.instance.getAllItems(Job).find { proj ->
        return (proj.fullName == buildName)
    }
    return build.lastStableBuild
}

@NonCPS
def getLastSuccessfulCommit(branch) {
    def build = getLastSuccessfulBuild(branch)
    if (build) {
        def actions = build.actions
        def gitAction = actions.find { action -> action instanceof hudson.plugins.git.util.BuildData && action.remoteUrls[0].contains('ucv-core') }
        return gitAction?.lastBuiltRevision?.sha1String
    }
    return null
}

def sendTestResult(resultFile, type, isPr) {
    if (fileExists(resultFile) && !isPr) {
        try {
            withCredentials([
                string(credentialsId: 'VELOCITY_DOG_FOOD_TENANT', variable: 'VELOCITY_DOG_FOOD_TENANT'),
            ]) {
                step([
                    $class: 'UploadMetricsFile',
                    debug: true,
                    fatal: false,
                    filePath: "${resultFile}",
                    pluginType: 'junitXML',
                    dataFormat: 'junitXML',
                    tenantId: "$VELOCITY_DOG_FOOD_TENANT",
                    appName: 'UCV',
                    environment: "CI${isPr ? ' PR' : ''}",
                    name: "${type} Tests",
                    testSetName: "${type}",
                    buildUrl: "${env.BUILD_URL}",
                    buildId: "${currentBuild.displayName}"
                ])
            }
        } catch (err) {
            print err
        }
    } else {
        println "File ${resultFile} does not exist, not uploading to Velocity."
    }
}

def saveTestResults(uploadToVelocity, isPr, addPackageName, tests) {
    tests.each { test ->
        println "Saving test results for '${test.title}'..."
        if (addPackageName && fileExists(test.path)) {
            def suitesExist = false
            try {
                suitesExist = sh(
                    script: "cat ${test.path} | grep '<testsuites' || echo ''",
                    returnStdout: true
                ).trim()
            } catch (err) { // groovylint-disable-line EmptyCatchBlock
            // swallow
            }
            def updatePath = '/testsuite/testcase/@classname'
            if (suitesExist) {
                updatePath = "/testsuites${updatePath}" // for multi-suite tests
            }
            sh "xmlstarlet edit --inplace --update '${updatePath}' --expr 'concat(\"${test.title}.\", .)' ${test.path}" // update package name for easier distinction between tests
        }
        try {
            archiveArtifacts artifacts: test.path
            junit testResults: test.path
            if (uploadToVelocity) {
                sendTestResult(test.path, test.title, isPr)
            }
        } catch (err) {
            println "Error saving test result for '${test.title}':"
            println err
            if (isBuildSucceeding()) {
                println 'build is currently succeeding, so marking as unstable'
                currentBuild.result = hudson.model.Result.UNSTABLE.toString()
            } else {
                println "build is currently '${currentBuild.currentResult}', so not changing build result"
            }
        }
    }
}

def runDownstreamTests(jobName, description, wait, publishResults, isPr, params, resultsArray, resultsDir, propagateResult = false, resultsFilter = '**/*.xml') {
    def downstreamBuild = build(
        job: jobName,
        parameters: params,
        wait: wait,
        quietPeriod: 0,
        propagate: propagateResult
    )
    if (wait) {
        def descriptionLink = "<a href='${downstreamBuild.getAbsoluteUrl()}' target='_blank'>${description}</a>"
        if (currentBuild.description) {
            currentBuild.description += ", ${descriptionLink}"
        } else {
            currentBuild.description = descriptionLink
        }
        if (resultsArray.size() > 0) {
            copyArtifacts(
                projectName: jobName,
                selector: specific(downstreamBuild.number.toString()),
                filter: resultsFilter,
                flatten: true,
                target: resultsDir
            )
            dir(resultsDir) {
                sh 'touch *.xml'
            }
            // clone so don't modify original values
            def appendedPathResultsArray = resultsArray.clone().collect { result ->
                def newResult = result.clone()
                newResult.path = "${resultsDir}/${result.path}"
                return newResult
            }
            saveTestResults(publishResults, isPr, false, appendedPathResultsArray)
        }
    }
}
def killOldBuilds(jobName) {
    def prev = currentBuild?.getPreviousBuild()
    for (int i = 0; i < 5 && prev != null; i++) {
        if (prev?.getRawBuild()?.isBuilding()) {
            println "Stopping previous build ${prev.getDisplayName()}"
            prev?.getRawBuild().doStop()
        }
        prev = prev?.getPreviousBuild()
    }
    def fourHoursAgo = System.currentTimeMillis() - (5 * 60 * 60 * 1000)
    def projName = currentBuild?.getFullProjectName()
    if (projName) {
        def currentPr = projName[projName.lastIndexOf('/') + 1..-1]
        def lastTest = Jenkins.instance.getItemByFullName(jobName)?.getLastBuild()
        while (lastTest && lastTest.getTimeInMillis() > fourHoursAgo) {
            prName = lastTest?.getEnvironment()['branchName']
            if (prName == currentPr) {
                if (lastTest?.isBuilding()) {
                    println "Stopping previous build ${lastTest.getDisplayName()}"
                    lastTest.doStop()
                }
            }
            lastTest = lastTest?.getPreviousBuild()
        }
    }
}
