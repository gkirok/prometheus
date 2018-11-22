def label = "${UUID.randomUUID().toString()}"
def V3IO_TSDB_VERSION = "v0.8.2"
def BUILD_FOLDER = '/go'
def github_user = "gkirok"
def docker_user = "gallziguazio"


properties([pipelineTriggers([[$class: 'PeriodicFolderTrigger', interval: '2m']])])
stage('release') {
    def TAG_VERSION = sh(
            script: "echo ${TAG_NAME} | tr -d '\n' | egrep '^v[\\.0-9]*-v[\\.0-9]*\$'",
            returnStdout: true
    ).trim()
    if ( TAG_VERSION ) {
        podTemplate(label: "prometheus-${label}", inheritFrom: 'kube-slave-dood') {
            node("prometheus-${label}") {
                withCredentials([
                        usernamePassword(credentialsId: '4318b7db-a1af-4775-b871-5a35d3e75c21', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME'),
                        secretText(credentialsId: 'dd7f75c5-f055-4eb3-9365-e7d04e644211', secretText: 'GIT_TOKEN')
                ]) {
                    stage('get release') {
                        // sh "curl https://api.github.com/repos/gkirok/prometheus/releases/latest"
                        sh """
                            curl -H "Authorization: bearer ${GIT_TOKEN}" -X POST -d '{"query": "query { repository(owner:\"gkirok\", name:\"prometheus\") { releases(last: 5) { nodes { tag { name } } } } }"' https://api.github.com/graphql
                        """
                    }

                    stage('prepare sources') {
                        sh """ 
                            cd ${BUILD_FOLDER}
                            pwd
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${github_user}/prometheus.git src/github.com/prometheus/prometheus
                            cd ${BUILD_FOLDER}/src/github.com/prometheus/prometheus
                            rm -rf vendor/github.com/v3io/v3io-tsdb/
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${github_user}/v3io-tsdb.git vendor/github.com/v3io/v3io-tsdb
                            cd vendor/github.com/v3io/v3io-tsdb
                            git checkout ${V3IO_TSDB_VERSION}
                            rm -rf .git vendor/github.com/prometheus
                        """
                    }

                    def V3IO_PROM_VERSION = sh(
                            script: "cat ${BUILD_FOLDER}/src/github.com/prometheus/prometheus/VERSION",
                            returnStdout: true
                    ).trim()

                    stage('build in dood') {
                        container('docker-cmd') {
                            sh """
                                cd ${BUILD_FOLDER}/src/github.com/prometheus/prometheus
                                docker build . -t ${docker_user}/v3io-prom:${V3IO_PROM_VERSION}-${V3IO_TSDB_VERSION} -f Dockerfile.multi
                            """
                            withDockerRegistry([credentialsId: "472293cc-61bc-4e9f-aecb-1d8a73827fae", url: ""]) {
                                sh "docker push ${docker_user}/v3io-prom:${V3IO_PROM_VERSION}-${V3IO_TSDB_VERSION}"
                            }
                        }
                    }

                    stage('git push') {
                        try {
                            sh """
                                git config --global user.email '${GIT_USERNAME}@iguazio.com'
                                git config --global user.name '${GIT_USERNAME}'
                                cd ${BUILD_FOLDER}/src/github.com/prometheus/prometheus;
                                git add vendor/github.com/v3io/v3io-tsdb;
                                git commit -am 'Updated TSDB to ${V3IO_TSDB_VERSION}';
                                git push origin master
                            """
                        } catch (err) {
                            echo "Can not push code to git"
                        }
                    }
                }
            }
        }
    }
}