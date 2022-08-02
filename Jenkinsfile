#!groovy

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
                       [field: '$.artifact.release', expectedValue: '^f37$']
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

                        if (msg['artifact']['builds'].size() > 10) {
                            echo "There are way too many (${msg['artifact']['builds'].size()} > 10) builds in the update. Skipping..."
                            return
                        }

                        msg['artifact']['builds'].each { build ->
                            allTaskIds.add(build['task_id'])
                        }

                        def testProfile = msg['artifact']['release']

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
