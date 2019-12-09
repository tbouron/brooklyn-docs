/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

node(label: 'ubuntu') {
    properties([
        pipelineTriggers([
            issueCommentTrigger('.*test this please.*')
        ])
    ])

    catchError {
        def environmentDockerImage

        def dockerTag = env.BUILD_TAG.replace('%2F', '-')

        withEnv(["DOCKER_TAG=${dockerTag}"]) {
            stage('Clone repository') {
                checkout scm
            }

            stage('Prepare environment') {
                echo 'Building docker image for test environment ...'
                environmentDockerImage = docker.build('brooklyn:${DOCKER_TAG}')
            }

            stage('Run tests') {
                environmentDockerImage.inside('-i --name brooklyn-${DOCKER_TAG} -v ${WORKSPACE}:/usr/build -w /usr/build') {
                    sh 'npm i'
                    sh 'npm run book'
                }
            }

            // Conditional stage to deploy artifacts, when not building a PR
            if (env.CHANGE_ID == null) {
                stage('Deploy documentation') {
                    sh 'git clone '
                    environmentDockerImage.inside('-i --name brooklyn-${DOCKER_TAG} -v ${WORKSPACE}:/usr/build -w /usr/build') {
                        sh 'GIT_URL=$(git config --get remote.origin.url)'
                        sh 'VERSION=$(cat book.json | jq -r .variables.brooklyn_version)'
                        sh 'git clone --single-branch --branch live $GIT_URL /tmp/docs'
                        sh 'rm -rf /tmp/docs/v/$VERSION/*'
                        sh 'cp -R ./_book/* /tmp/docs/v/$VERSION'
                        sh 'cd /tmp/docs'
                        sh 'git push origin live'
                    }
                }
            }
        }
    }

    // Conditional stage, when not building a PR
    if (env.CHANGE_ID == null) {
        stage('Send notifications') {
            // Send email notifications
            step([
                $class: 'Mailer',
                notifyEveryUnstableBuild: true,
                recipients: 'dev@brooklyn.apache.org',
                sendToIndividuals: false
            ])
        }
    }
}
