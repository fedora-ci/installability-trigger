#!groovy

// koji-build: trigger if the dist tag matches this regex
def triggerDistTagRegEx = '^f[3-9]{1}[0-9]{1}$'

// If this is a pull request, run these tests and quit
if (env.CHANGE_ID) {
    assert 'f42' ==~ triggerDistTagRegEx
    assert 'f59' ==~ triggerDistTagRegEx

    currentBuild.result = 'SUCCESS'
    echo('All tests passed!')
    return
}


def msg
def artifactId
def releaseId
def additionalArtifactIds
def allTaskIds = [] as Set

pipeline {

    agent none

    options {
        buildDiscarder(logRotator(daysToKeepStr: '3', artifactNumToKeepStr: '100'))
        skipDefaultCheckout()
    }

    triggers {
       ciBuildTrigger(
           noSquash: true,
           providerList: [
               rabbitMQSubscriber(
                   name: 'RabbitMQ',
                   overrides: [
                       topic: 'org.fedoraproject.prod.bodhi.update.status.testing.koji-build-group.build.complete',
                       queue: 'osci-pipelines-queue-14'
                   ],
                   checks: [
                       [field: '$.update.release.dist_tag', expectedValue: '^(f[3-9]{1}[0-9]{1}|epel10.[0-9]+)$'],
                       [field: '$.update.release.branch', expectedValue: '^(f[3-9]{1}[0-9]{1}|rawhide|epel10)$']
                   ]
               )
           ]
       )
    }

    parameters {
        string(name: 'CI_MESSAGE', defaultValue: '{}', description: 'CI Message')
    }

    stages {
        stage('Trigger Testing') {
            steps {
                script {
                    msg = readJSON text: CI_MESSAGE

                    if (msg) {

                        if (msg['update']['builds'].size() > 20) {
                            echo "There are way too many (${msg['update']['builds'].size()} > 20) builds in the update. Skipping..."
                            return
                        }

                        msg['artifact']['builds'].each { build ->
                            allTaskIds.add(build['task_id'])
                        }

                        if (msg['update']['release']['branch'] == 'epel10') {
                            def testProfile = 'epel10'
                        } else {
                            def testProfile = msg['update']['release']['dist_tag']
                        }

                        if (allTaskIds) {
                            allTaskIds.each { taskId ->
                                artifactId = "koji-build:${taskId}"
                                additionalArtifactIds = allTaskIds.findAll{ it != taskId }.collect{ "koji-build:${it}" }.join(',')

                                build(
                                    job: 'fedora-ci/installability-pipeline/master',
                                    wait: false,
                                    parameters: [
                                        string(name: 'ARTIFACT_ID', value: artifactId),
                                        string(name: 'ADDITIONAL_ARTIFACT_IDS',value: additionalArtifactIds),
                                        string(name: 'TEST_PROFILE',value: testProfile)
                                    ]
                                )
                            }
                        }
                    }
                }
            }
        }
    }
}
