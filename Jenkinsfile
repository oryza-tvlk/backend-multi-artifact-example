#!/usr/bin/env groovy

@Library("backend-jenkins")
import com.traveloka.common.jenkins.beiartf.BeiartfUtil

node("aws-ecs-small") {
    wrap([$class: 'TimestamperBuildWrapper']) {
        def global_pipeline
        def TEST_TIER = "mediumTest"

        configFileProvider([configFile(fileId: 'global-pipeline-groovy', targetLocation: 'global_pipeline.groovy')]) {
            global_pipeline = load 'global_pipeline.groovy'
        }

        global_pipeline.phabricatorCreateArtifact(PHID)
        try {
            stage('Checkout') {
                checkout scm
            }

            stage('Build') {
                withCredentials([
                    [
                        $class           : 'AmazonWebServicesCredentialsBinding',
                        accessKeyVariable: "AWS_ACCESS_KEY_ID",
                        credentialsId    : 'tvlk-dev-user-jenkins',
                        secretKeyVariable: "AWS_SECRET_ACCESS_KEY"
                    ]
                ]) {
                    BeiartfUtil.assumeRole(this)
                    sh "./gradlew jar"
                }
            }

            stage('Test') {
                withCredentials([
                    [
                        $class           : 'AmazonWebServicesCredentialsBinding',
                        accessKeyVariable: "AWS_ACCESS_KEY_ID",
                        credentialsId    : 'tvlk-dev-user-jenkins',
                        secretKeyVariable: "AWS_SECRET_ACCESS_KEY"
                    ]
                ]) {
                    try {
                        BeiartfUtil.assumeRole(this)
                        sh "./gradlew ${TEST_TIER}AndDeploy"
                        putReportUrl(JOB_NAME, TEST_TIER)
                    } catch (Exception e) {
                        putReportUrl(JOB_NAME, TEST_TIER)
                        throw e
                    }
                }
            }
            global_pipeline.phabricatorSucceed(PHID)
        } catch (Exception e) {
            global_pipeline.phabricatorFailed(PHID)
            throw e
        }
        stash name: "$JOB_NAME-$BUILD_NUMBER", useDefaultExcludes: false
    }
}

def userInput
timeout(time: 1, unit: 'HOURS') {
    userInput = input message: 'Publish to artifactory?', parameters: [string(defaultValue: 'minor', description: '''which semantic version number to increase (major, minor, patch)
''', name: 'RELEASE_SCOPE')], submitterParameter: 'PUBLISHER'
}

node("aws-ecs-small") {
    stage('Publish') {
        unstash "$JOB_NAME-$BUILD_NUMBER"
        sshagent(['jenkins-master-phabricator-pushable']) {
            withCredentials([
              [
                $class           : 'AmazonWebServicesCredentialsBinding',
                accessKeyVariable: "AWS_ACCESS_KEY_ID",
                credentialsId    : 'tvlk-dev-user-jenkins',
                secretKeyVariable: "AWS_SECRET_ACCESS_KEY"
              ]
            ]) {
                BeiartfUtil.assumeRole(this)
                sh "mkdir ~/.ssh && ssh-keyscan git.traveloka.com >> ~/.ssh/known_hosts"
                sh "./gradlew final -Prelease.scope=${userInput['RELEASE_SCOPE']}"
            }
        }
    }
    stage('Label') {
        def VERSION = sh script: "git describe --tags --always", returnStdout: true
        currentBuild.displayName = "#$BUILD_NUMBER - $VERSION"
        currentBuild.description = "published with ${userInput['PUBLISHER']} approval"
    }
}

void putReportUrl(project_name, test_tier) {
    def file_path = "/tmp/$project_name/$test_tier-report-url"
    if(fileExists(file_path)){
        def report_url = readFile(file_path)
        currentBuild.description += " <a href=\"$report_url\">$test_tier</a> "
    }
}
