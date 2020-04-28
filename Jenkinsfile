pipeline {
    agent none
    options {
        ansiColor("xterm")
        timeout(time: 6, unit: "HOURS")
        timestamps()
    }
    parameters {
        choice(name: "BUILD_TYPE", choices: ["Release", "Debug"], description: "")
        choice(name: "CHANNEL", choices: ["nightly", "dev", "beta", "release", "development"], description: "")
        string(name: "SLACK_BUILDS_CHANNEL", defaultValue: "#build-downloads-bot", description: "The Slack channel to send the list of artifact download links to. Leave blank to skip sending the message.")
        booleanParam(name: "SKIP_SIGNING", defaultValue: true, description: "")
        booleanParam(name: "WIPE_WORKSPACE", defaultValue: false, description: "")
        booleanParam(name: "SKIP_INIT", defaultValue: false, description: "")
        booleanParam(name: "DISABLE_SCCACHE", defaultValue: false, description: "")
        booleanParam(name: "DCHECK_ALWAYS_ON", defaultValue: true, description: "")
        booleanParam(name: "DEBUG", defaultValue: false, description: "")
    }
    environment {
        GITHUB_CREDENTIAL_ID = "brave-builds-github-token-for-pr-builder"
        REFERRAL_API_KEY = credentials("REFERRAL_API_KEY")
        BRAVE_SERVICES_KEY = credentials("brave-services-key")
        BRAVE_INFURA_PROJECT_ID = credentials("brave-infura-project-id")
        BRAVE_GOOGLE_API_KEY = credentials("npm_config_brave_google_api_key")
        BRAVE_ARTIFACTS_S3_BUCKET = credentials("brave-jenkins-artifacts-s3-bucket")
        SLACK_USERNAME_MAP = credentials("github-to-slack-username-map")
        SIGN_WIDEVINE_PASSPHRASE = credentials("447b2fa7-c989-43af-9047-8ae158fad0a3")
        BINANCE_CLIENT_ID = credentials("binance-client-id")
    }
    stages {
        stage("env") {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIAL_ID}", usernameVariable: "PR_BUILDER_USER", passwordVariable: "PR_BUILDER_TOKEN")]) {
                    script {
                        setEnv()
                    }
                }
            }
        }
        stage("abort") {
            steps {
                script {
                    checkAndAbortBuild()
                }
            }
        }
        stage("s3-init") {
            agent { label "master" }
            steps {
                sh "echo ${BUILD_URL} > build.txt"
                s3Upload(bucket: BRAVE_ARTIFACTS_S3_BUCKET, path: BUILD_TAG_SLASHED, includePathPattern: "build.txt")
            }
        }
        stage("build-all") {
            agent { label "master" }
            when {
                beforeAgent true
                expression { !SKIP }
            }
            steps {
                startBraveBrowserBuild()
            }
        }
    }
    post {
        always {
            script {
                if (env.SLACK_USERNAME) {
                    def slackColorMap = ["SUCCESS": "good", "FAILURE": "danger", "UNSTABLE": "warning", "ABORTED": null]
                    slackSend(color: slackColorMap[currentBuild.currentResult], channel: env.SLACK_USERNAME, message: "[${BUILD_TAG_SLASHED} `${BRANCH}`] " + currentBuild.currentResult + " (<${BUILD_URL}/flowGraphTable/?auto_refresh=true|Open>)")
                }
            }
            node("master") {
                script {
                    if (SLACK_BUILDS_CHANNEL) {
                        sendSlackDownloadsNotification()
                    }
                }
            }
        }
    }
}

def setEnv() {
    BUILD_TYPE = params.BUILD_TYPE
    CHANNEL = params.CHANNEL
    SLACK_BUILDS_CHANNEL = params.SLACK_BUILDS_CHANNEL
    DCHECK_ALWAYS_ON = params.DCHECK_ALWAYS_ON
    CHANNEL_CAPITALIZED = CHANNEL.equals("release") ? "" : CHANNEL.capitalize()
    CHANNEL_CAPITALIZED_BACKSLASHED_SPACED = CHANNEL.equals("release") ? "" : "\\ " + CHANNEL.capitalize()
    SKIP_SIGNING = params.SKIP_SIGNING ? "--skip_signing" : ""
    WIPE_WORKSPACE = params.WIPE_WORKSPACE ? "WipeWorkspace" : "RelativeTargetDirectory"
    SKIP_INIT = params.SKIP_INIT
    DISABLE_SCCACHE = params.DISABLE_SCCACHE
    DEBUG = params.DEBUG
    OUT_DIR = "src/out/" + BUILD_TYPE
    BUILD_TAG_SLASHED = env.JOB_NAME + "/" + env.BUILD_NUMBER
    LINT_BRANCH = "TEMP_LINT_BRANCH_" + env.BUILD_NUMBER
    BRAVE_GITHUB_TOKEN = "brave-browser-releases-github"
    GITHUB_API = "https://api.github.com/repos/brave"
    SKIP = false
    SKIP_ANDROID = false
    SKIP_IOS = false
    SKIP_LINUX = false
    SKIP_MACOS = false
    SKIP_WINDOWS = false
    RUN_NETWORK_AUDIT = false
    BRANCH = env.BRANCH_NAME
    BASE_BRANCH = "master"
    BRAVE_CORE_BRANCH = "master"
    if (env.CHANGE_BRANCH) {
        BRANCH = env.CHANGE_BRANCH
        BASE_BRANCH = env.CHANGE_TARGET
        BRAVE_CORE_BRANCH = BASE_BRANCH
        def bbPrNumber = readJSON(text: httpRequest(url: GITHUB_API + "/brave-browser/pulls?head=brave:" + BRANCH, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)[0].number
        def bbPrDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-browser/pulls/" + bbPrNumber, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)
        SKIP = bbPrDetails.mergeable_state.equals("draft") || bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip") }.equals(1)
        SKIP_ANDROID = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-android") }.equals(1)
        SKIP_IOS = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-ios") }.equals(1)
        SKIP_LINUX = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-linux") }.equals(1)
        SKIP_MACOS = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-macos") }.equals(1)
        SKIP_WINDOWS = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-windows") }.equals(1)
        RUN_NETWORK_AUDIT = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/run-network-audit") }.equals(1)
        env.SLACK_USERNAME = readJSON(text: SLACK_USERNAME_MAP)[bbPrDetails.user.login]
        env.BRANCH_PRODUCTIVITY_HOMEPAGE = "https://github.com/brave/brave-browser/pull/${bbPrNumber}"
        env.BRANCH_PRODUCTIVITY_NAME = "Brave Browser PR #${bbPrNumber}"
        env.BRANCH_PRODUCTIVITY_DESCRIPTION = bbPrDetails.title
        env.BRANCH_PRODUCTIVITY_USER = bbPrDetails.user.login
    }
    BRANCH_EXISTS_IN_BC = httpRequest(url: GITHUB_API + "/brave-core/branches/" + BRANCH, validResponseCodes: "100:499", customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).status.equals(200)
    if (BRANCH_EXISTS_IN_BC) {
        def bcPrDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-core/pulls?head=brave:" + BRANCH, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)[0]
        if (bcPrDetails) {
            env.BC_PR_NUMBER = bcPrDetails.number
            BRAVE_CORE_BRANCH = BRANCH
            bcPrDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-core/pulls/" +  env.BC_PR_NUMBER, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)
            BASE_BRANCH = bcPrDetails.base.ref
            SKIP = bcPrDetails.mergeable_state.equals("draft") || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip") }.equals(1)
            SKIP_ANDROID = SKIP_ANDROID || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-android") }.equals(1)
            SKIP_IOS = SKIP_IOS || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-ios") }.equals(1)
            SKIP_LINUX = SKIP_LINUX || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-linux") }.equals(1)
            SKIP_MACOS = SKIP_MACOS || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-macos") }.equals(1)
            SKIP_WINDOWS = SKIP_WINDOWS || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-windows") }.equals(1)
            RUN_NETWORK_AUDIT = RUN_NETWORK_AUDIT || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/run-network-audit") }.equals(1)
            env.SLACK_USERNAME = readJSON(text: SLACK_USERNAME_MAP)[bcPrDetails.user.login]
            env.BRANCH_PRODUCTIVITY_HOMEPAGE = "https://github.com/brave/brave-core/pull/${bcPrDetails.number}"
            env.BRANCH_PRODUCTIVITY_NAME = "Brave Core PR #${bcPrDetails.number}"
            env.BRANCH_PRODUCTIVITY_DESCRIPTION = bcPrDetails.title
            env.BRANCH_PRODUCTIVITY_USER = bcPrDetails.user.login
        }
    }
    if (env.SLACK_USERNAME) {
        slackSend(color: null, channel: env.SLACK_USERNAME, message: "[${BUILD_TAG_SLASHED} `${BRANCH}`] STARTED (<${BUILD_URL}/flowGraphTable/?auto_refresh=true|Open>)")
    }
}

def checkAndAbortBuild() {
    if (SKIP) {
        echo "Aborting build as PR is in draft or has \"CI/skip\" label"
        stopCurrentBuild()
    }
    else if (BRANCH_EXISTS_IN_BC) {
        if (isStartedManually()) {
            if (env.BC_PR_NUMBER) {
                echo "Aborting build as PR exists in brave-core and build has not been started from there"
                echo "Use " + env.JENKINS_URL + "view/ci/job/brave-core-build-pr/view/change-requests/job/PR-" + env.BC_PR_NUMBER + " to trigger"
            }
            else {
                echo "Aborting build as there's a matching branch in brave-core, please create a PR there first"
                echo "Use https://github.com/brave/brave-core/compare/" + BASE_BRANCH + "..." + BRANCH + " to create PR"
            }
            SKIP = true
            stopCurrentBuild()
        }
    }
    if (!SKIP) {
        for (build in getBuilds()) {
            if (build.isBuilding() && build.getNumber() < env.BUILD_NUMBER.toInteger()) {
                echo "Aborting older running build " + build
                build.doStop()
                // build.finish(hudson.model.Result.ABORTED, new java.io.IOException("Aborting build"))
            }
        }
        sleep(time: 1, unit: "MINUTES")
    }
}

def sendSlackDownloadsNotification() {
    // Notify links to all the build files
    echo "Reading all uploaded files for slack notification..."
    def attachments = getSlackFileAttachments()
    if (!attachments.isEmpty()) {
        def messageText = getSlackMessageText()
        echo "Sending builds message: '${messageText}'."
        slackSend(channel: SLACK_BUILDS_CHANNEL, message: messageText, attachments: attachments)
    } else {
        echo "Not sending message, no files found."
    }
}

def getSlackFileAttachments () {
    def files = s3FindFiles(bucket: BRAVE_ARTIFACTS_S3_BUCKET, path: BUILD_TAG_SLASHED, glob: "**")
    def attachments = [ ]
    files.each { file ->
        echo "Found file: ${file.name}"
        if (!file.directory && file.name != "build.txt") {
            attachments.add([
                title: file.name,
                title_link: "https://" + BRAVE_ARTIFACTS_S3_BUCKET + ".s3.amazonaws.com/" + BUILD_TAG_SLASHED.replace('%2F', '%252F') + "/" + file.path,
                footer: byteLengthToString(file.length)
            ])
        }
    }
    return attachments
}

def getSlackMessageText () {
    def messageText = "Downloads are available for branch `${BRANCH}`"
    if (env.BRANCH_PRODUCTIVITY_NAME) {
        messageText += "\n(<${env.BRANCH_PRODUCTIVITY_HOMEPAGE}|${env.BRANCH_PRODUCTIVITY_NAME}>)"
    }
    if (env.SLACK_USERNAME) {
        messageText += " by <${env.SLACK_USERNAME}>"
    }
    else if (env.BRANCH_PRODUCTIVITY_USER) {
        messageText += " by ${env.BRANCH_PRODUCTIVITY_USER}"
    }
    if (env.BRANCH_PRODUCTIVITY_DESCRIPTION) {
        messageText += "\n_${env.BRANCH_PRODUCTIVITY_DESCRIPTION}_"
    }
    return messageText
}

def byteLengthToString (long byteLength) {
    def base = 1048576L
    def mbLength = (byteLength / base) as Double
    return mbLength.round() + " Mb"
}

@NonCPS
def stopCurrentBuild() {
    Jenkins.instance.getItemByFullName(env.JOB_NAME).getLastBuild().doStop()
}

@NonCPS
def isStartedManually() {
    return Jenkins.instance.getItemByFullName(env.JOB_NAME).getLastBuild().getCause(hudson.model.Cause$UpstreamCause) == null
}

@NonCPS
def getBuilds() {
    return Jenkins.instance.getItemByFullName(env.JOB_NAME).builds
}

def startBraveBrowserBuild() {
    jobDsl(scriptText: 'job("' + "brave-browser-build-pr-${BRANCH}" + '")')
    params = [
        string(name: "BUILD_TYPE", value: BUILD_TYPE),
        string(name: "CHANNEL", value: CHANNEL),
        string(name: "SLACK_BUILDS_CHANNEL", value: SLACK_BUILDS_CHANNEL),
        booleanParam(name: "SKIP_SIGNING", value: SKIP_SIGNING),
        booleanParam(name: "WIPE_WORKSPACE", value: WIPE_WORKSPACE),
        booleanParam(name: "SKIP_INIT", value: SKIP_INIT),
        booleanParam(name: "DISABLE_SCCACHE", value: DISABLE_SCCACHE),
        booleanParam(name: "DCHECK_ALWAYS_ON", value: DCHECK_ALWAYS_ON),
        booleanParam(name: "DEBUG", value: DEBUG)
    ]
    currentBuild.result = build(job: "brave-browser-build-pr-${BRANCH}", parameters: params, propagate: false).result
}
