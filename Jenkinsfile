#!groovy

import groovy.json.JsonOutput
import java.util.Optional
import java.text.SimpleDateFormat
import hudson.model.*
import jenkins.model.*
import hudson.model.Fingerprint.RangeSet

node('agent-gorela') {
// node {
    def nodeEnv = "production"
    def nodeEnvForClient = "production"
    def scmVars
    def currentBranch
    def dockerTag
    def gitAuthour = "gorela"
    def gitLastCommitMessage = "init"
    def namespace = "gorela"
    def kibanaurl = "https://staging-kibana.gorelas.com/goto/7b48edc6002cb9e3ecdb7eb4ab1e9dca"
    String[] buildTargetServerList = [
        "presenter", "interlocker", "admin",
        "allocator", "admin-client", "client-logger",
        "redoc-order-agency", "redoc-delivery-agency", "store-socket-server",
        "rider-location-consumer", "rider-location-sender", "scheduler-server",
        "rider-location-reader", "admin-socket-server", "data-masking-worker", "data-masking-distributer", "store-event-consumer"
        ]

    try {
        scmVars = checkout scm

        // git tag 동기화: git tag 는 git pull 로 remote 에서 제거된 tag 가 제거 되지 않음. tag 는 jenkins 에서 알아서 fetch 함!
        sh(script: "git checkout ${scmVars.GIT_BRANCH}")
        // sh(returnStdout: true, script: "git tag -l | xargs git tag -d") // 마스터 브랜치에서 태그 정보를 가져 오지 못함

        println "current build number => " + env.BUILD_NUMBER

        // build 대상 서버 commit message 에서 추출
        def isBuildTargetExist = sh(returnStdout: true, returnStatus: true, script: 'git log -1 --pretty=%B | grep target-build')

        if (isBuildTargetExist == 0) {
            def targetBuildFromCommit = sh(returnStdout: true, script: 'git log -1 --pretty=%B | grep target-build').trim()
            def server = (targetBuildFromCommit =~ /\[target-build:.*\/]/)
            println "server ==>>" + server.size()

            if (server.size() > 0) {
                // [target-build: interlocker, allocator /] 이런식의 commit message 의 경우, 각 요소 앞 뒤 공백이 포함 되어 list 가 생성되게 됨
                buildTargetServerList = server[0].replace("[target-build:", "").replace("/]", "").split(",")
                buildTargetServerList = trimElementsInArray(buildTargetServerList)
                println "server ==>>" + buildTargetServerList
            }
        }

        currentBranch = "${scmVars.GIT_BRANCH}"
        println "currentBranch == ${currentBranch}"

        // [skip ci] commit message 에 포함 되면 build skip
        def isCommitMessageMatchBuildSkip = sh (returnStdout: true, returnStatus: true, script: "git log -1 --pretty=%B | grep '\\[skip build\\]'")
        if (isCommitMessageMatchBuildSkip == 0) {
            println "commit message match [skip build] => " + isCommitMessageMatchBuildSkip
            // deleteCurrentBuild()
            return
        }

        // feature branch 인 경우, [let it build] 또는 [let it build for qc] commit message 에 포함 안되면 build skip
        def isCommitMessageMatchLetItBuild = sh(returnStdout: true, returnStatus: true, script: "git log -1 --pretty=%B | grep '\\[let it build\\]'")
        def isCommitMessageMatchLetItBuildForQc = sh(returnStdout: true, returnStatus: true, script: "git log -1 --pretty=%B | grep '\\[let it build for qc\\]'")
        if (currentBranch.startsWith("origin/feature") || currentBranch.startsWith("feature/")) {
            if (isCommitMessageMatchLetItBuild == 1 && isCommitMessageMatchLetItBuildForQc == 1) {
                println "commit message not math [let it build] or [let it build for qc] => ${isCommitMessageMatchLetItBuild} / ${isCommitMessageMatchLetItBuildForQc}"
                // deleteCurrentBuild()
                return
            }
        }

        stage('Set Variable') {
            def commit = sh(returnStdout: true, script: 'git rev-parse HEAD')
            gitAuthour = sh(returnStdout: true, script: "git --no-pager show -s --format='%an' ${commit}").trim()
            gitLastCommitMessage = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()

            // production, staging, develop 환경 별 분기
            if (currentBranch == "origin/master" || currentBranch == "master") {
                dockerTag = sh(returnStdout: true, script: "git tag --sort version:refname | tail -1").trim()
                println "master dockerTag = ${dockerTag}"

            } else if (currentBranch.startsWith("origin/release") || currentBranch.startsWith("release/")) {
                gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim().take(6)
                // dockerTag = "staging-${currentBranch.split('/')[2]}-${gitCommit}"
                dockerTag = "release-${gitCommit}"
                nodeEnv = "staging"
                nodeEnvForClient = "staging"
                namespace = "gorela-staging"
                println "release dockerTag = ${dockerTag}"
            } else if (currentBranch.startsWith("origin/hotfix") || currentBranch.startsWith("hotfix/")) {
                gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim().take(6)
                // dockerTag = "staging-${currentBranch.split('/')[2]}-${gitCommit}"
                dockerTag = "hotfix-${gitCommit}"
                nodeEnv = "staging"
                nodeEnvForClient = "staging"
                namespace = "gorela-staging"
                println "hotfix dockerTag = ${dockerTag}"
            } else if (currentBranch == "origin/develop" || currentBranch == "develop") {
                gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim().take(6)
                dockerTag = "develop-${gitCommit}"
                nodeEnv = "development"
                nodeEnvForClient = "development"
                namespace = "gorela-dev"
                kibanaurl = "https://staging-kibana.gorelas.com/goto/f29b76e2fe3e4ad16aaffa2c0325ec01"
                println "develop dockerTag = ${dockerTag}"
            } else {
                gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim().take(6)
                dockerTag = "other-branch-${gitCommit}"
                nodeEnv = "development"
                nodeEnvForClient = "development"
                namespace = "gorela-dev"
                println "other-branch dockerTag = ${dockerTag}"

                if (isCommitMessageMatchLetItBuildForQc == 0) {
                    namespace = "gorela-qc"
                    nodeEnvForClient = "qc"
                }

                return
            }
        }

        stage('Build start send to slack') {
            notifySlack("", [
                [
                    title: "[${env.JOB_NAME}] [${nodeEnv}] ${currentBranch} by ${gitAuthour} BUILD START!",
                    title_link: "${env.BUILD_URL}",
                    color: "#3498DB",
                    author_name: "${gitAuthour}",
                    fields: [
                        [
                            title: "Branch",
                            value: "${currentBranch}",
                            short: false
                        ],
                        [
                            title: "Last Commit message",
                            value: "${gitLastCommitMessage}",
                            short: false
                        ]
                    ],
                    footer: "${env.JOB_NAME} - ${nodeEnv}",
                    ts: System.currentTimeMillis() / 1000
                ]
            ])
        }

        stage('Build Push') {
            for (int i = 0; i < buildTargetServerList.size(); i++) {
                def targetSvr = buildTargetServerList[i]
                stage("${targetSvr}") {
                    if ( targetSvr == 'admin-client' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host --build-arg NODE_ENV=${nodeEnvForClient} ./client/admin/")

                    } else if ( targetSvr == 'client-logger' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/${targetSvr}/")

                    } else if ( targetSvr == 'redoc-order-agency' ) {
                        if(namespace == "gorela-staging" || namespace == "gorela-dev") {
                            buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/interlocker/src/public/swagger/orderAgency/")
                        } else return
                    } else if ( targetSvr == 'redoc-delivery-agency') {
                        if(namespace == "gorela-staging" || namespace == "gorela-dev") {
                            buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/interlocker/src/public/swagger/deliveryAgency/")
                        } else return
                    } else if ( targetSvr == 'store-event-consumer' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/ -f ./server/${targetSvr}/Dockerfile")

                    } else if ( targetSvr == 'store-socket-server' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/ -f ./server/${targetSvr}/Dockerfile")

                    } else if ( targetSvr == 'admin-socket-server' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/ -f ./server/${targetSvr}/Dockerfile")

                    } else if ( targetSvr == 'rider-location-consumer' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/${targetSvr}/")

                    } else if ( targetSvr == 'rider-location-sender' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/ -f ./server/${targetSvr}/Dockerfile")

                    } else if ( targetSvr == 'rider-location-reader' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/${targetSvr}/")

                    } else if ( targetSvr == 'scheduler-server' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/ -f ./server/${targetSvr}/Dockerfile")

                    } else if ( targetSvr == 'data-masking-distributer' ) {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host ./server/ -f ./server/${targetSvr}/Dockerfile")

                    } else {
                        buildApp = docker.build("gorela/${targetSvr}:${dockerTag}", "--network host --build-arg serverName=${targetSvr} ./server/")
                    }

                    docker.withRegistry("https://653983231979.dkr.ecr.ap-northeast-2.amazonaws.com", "ecr:ap-northeast-2:jenkins") {
                        buildApp.push()
                    }
                }
            }
        }

        stage('Build success send to slack') {
            def deployCliList = []

            for (int i = 0; i < buildTargetServerList.size(); i++) {
                def targetSvr = buildTargetServerList[i]
                if((namespace == 'gorela' || namespace == 'gorela-qc') && (targetSvr == 'redoc-order-agency' || targetSvr == 'redoc-delivery-agency')){
                    continue
                }
                deployCliList.add([
                    title: "deploy to k8s for ${targetSvr}",
                    value: "```kubectl --record --namespace=${namespace} set image deployment/${targetSvr} ${targetSvr}=653983231979.dkr.ecr.ap-northeast-2.amazonaws.com/gorela/${targetSvr}:${dockerTag}```",
                    short: false
                ])
                // presenter인 경우, presenter-readonly deploy script 추가
                if (targetSvr == 'presenter') {
                    deployCliList.add([
                        title: "deploy to k8s for presenter-readonly",
                        value: "```kubectl --record --namespace=${namespace} set image deployment/presenter-readonly presenter-readonly=653983231979.dkr.ecr.ap-northeast-2.amazonaws.com/gorela/${targetSvr}:${dockerTag}```",
                        short: false
                    ])
                }
            }

            def notiFields = [
                [
                    title: 'Branch',
                    value: "${currentBranch}",
                    short: true
                ],
                [
                    title: 'Target Server',
                    value: "${buildTargetServerList}",
                    short: true
                ],
                [
                    title: 'Last Commit message',
                    value: "${gitLastCommitMessage}",
                    short: false
                ]
            ]

            if (!deployCliList.isEmpty()) {
                notiFields.add(deployCliList)
            }

            notiFields.add([
                [
                    title: 'check log from k8s pod',
                    value: "kubectl --namespace ${namespace} logs -f {pod name}",
                    short: false
                ],
                [
                    title: 'check log from kibana',
                    value: "${kibanaurl}",
                    short: false
                ]
            ])

            notifySlack("", [
                [
                    title: "[${env.JOB_NAME}] [${nodeEnv}] ${currentBranch} by ${gitAuthour} BUILD SUCCESS!",
                    title_link: "${env.BUILD_URL}",
                    color: "#1E8449",
                    author_name: "${gitAuthour}",
                    fields: notiFields.flatten(),
                    footer: "${env.JOB_NAME} - ${nodeEnv}",
                    ts: System.currentTimeMillis() / 1000
                ]
            ])
        }

    } catch (e) {
        notifySlack("", [
            [
                title: "[${env.JOB_NAME}] [${nodeEnv}] ${currentBranch} by ${gitAuthour} BUILD FAIL!",
                title_link: "${env.BUILD_URL}",
                color: "#CB4335",
                author_name: "${gitAuthour}",
                fields: [
                    [
                        title: "Branch",
                        value: "${currentBranch}",
                        short: false
                    ],
                    [
                        title: "Last Commit message",
                        value: "${gitLastCommitMessage}",
                        short: false
                    ]
                ],
                footer: "${env.JOB_NAME} - ${nodeEnv}",
                ts: System.currentTimeMillis() / 1000
            ]
        ])
        throw e
    }
}

def notifySlack(text, attachments) {
    def slackURL = 'https://hooks.slack.com/services/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' // gorela_ci_cd
    // def slackURL = 'https://hooks.slack.com/services/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'  // shared_alert_channel
    def jenkinsIcon = 'https://avatars.slack-edge.com/2019-05-08/628787263668_7a9ee5e84462be745c7a_48.jpg'

    def payload = JsonOutput.toJson([text: text,
        channel: 'gorela_ci_cd',
        // channel: 'shared_alert_channel',
        username: "Jenkins",
        icon_url: jenkinsIcon,
        attachments: attachments
    ])

    sh "curl -X POST --data-urlencode \'payload=${payload}\' ${slackURL}"
}

def deleteCurrentBuild() {
    def job = jenkins.model.Jenkins.instance.getItem("gorela")
    def buildRange = RangeSet.fromString(env.BUILD_NUMBER.toString(), true);
    job.getBuilds(buildRange).each { it.delete() }
}

/**
 * commit message [target-build: interlocker, allocator /] 를 가공 시
 * 각 요소가 띄어쓰기가 포함 되어 list 에 담겨 지지 않도록 처리하기 위한 함수
 */
def trimElementsInArray(buildTargetServerList) {
    def serverForBuildWithTrim = []

    if (buildTargetServerList.size() <= 0) {
        return serverForBuildWithTrim
    }

    for (int i=0; i<buildTargetServerList.size(); ++i) {
        String serverForBuild = buildTargetServerList[i].trim()

        serverForBuildWithTrim.add(serverForBuild)
    }


    return serverForBuildWithTrim
}
