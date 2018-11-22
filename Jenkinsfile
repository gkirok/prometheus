def label = "${UUID.randomUUID().toString()}"
def V3IO_TSDB_VERSION = "v0.8.2"
def BUILD_FOLDER = '/go'
def github_user = "gkirok"
def docker_user = "gallziguazio"



podTemplate(label: "prometheus-${label}", inheritFrom: 'kube-slave-dood') {
    node("prometheus-${label}") {
        withCredentials([
                usernamePassword(credentialsId: '4318b7db-a1af-4775-b871-5a35d3e75c21', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME'),
        ]) {
            stage('get release') {
                // sh "curl https://api.github.com/repos/gkirok/prometheus/releases/latest"
                sh """
                    curl -X POST -d " \\
                     { \\
                       \\"query\\": \\"query { viewer { login }}\\" \\
                     } \\
                    " https://api.github.com/graphql
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

            def V3IO_PROM_VERSION = sh (
                    script: "cat ${BUILD_FOLDER}/src/github.com/prometheus/prometheus/VERSION",
                    returnStdout: true
            ).trim()

            stage('build in dood') {
                container('docker-cmd') {
                    sh """
                        cd ${BUILD_FOLDER}/src/github.com/prometheus/prometheus
                        docker build . -t ${docker_user}/v3io-prom:${V3IO_PROM_VERSION}-${V3IO_TSDB_VERSION} -f Dockerfile.multi
                    """
                    withDockerRegistry([ credentialsId: "472293cc-61bc-4e9f-aecb-1d8a73827fae", url: "" ]) {
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