pipeline {
    agent any

    parameters {
        booleanParam(name: 'CLEAN_WORKSPACE', defaultValue: true, description: 'Clean workspace before starting the pipeline')
        string(name: 'JMETER_VERSION', defaultValue: '5.6.3', description: 'Apache JMeter version to download and use')
        string(name: 'JMX_FILE',        defaultValue: 'Thread Group.jmx', description: 'JMX test plan filename inside ApacheJmeterTest repo')
        string(name: 'PTETESTDEMO_BRANCH', defaultValue: 'main', description: 'Branch to checkout for ptetestdemo')
        string(name: 'APACHEJMETER_BRANCH', defaultValue: 'main', description: 'Branch to checkout for ApacheJmeterTest')
        booleanParam(name: 'ENABLE_INFLUXDB', defaultValue: true, description: 'Push JMeter time-series metrics to InfluxDB for Grafana')
        string(name: 'INFLUXDB_URL', defaultValue: 'http://host.docker.internal:8086', description: 'InfluxDB base URL (use host.docker.internal from Dockerized Jenkins on Mac/Win, or service name in compose)')
        booleanParam(name: 'INFLUXDB_SUMMARY_ONLY', defaultValue: false, description: 'If true, JMeter sends only end-of-test summary (no live time-series). Keep false for live Grafana metrics.')
        string(name: 'INFLUXDB_ORG', defaultValue: 'jmetertest', description: 'InfluxDB organization')
        string(name: 'INFLUXDB_BUCKET', defaultValue: 'telegraf', description: 'InfluxDB bucket name')
        string(name: 'INFLUXDB_APPLICATION', defaultValue: 'ptetest-jmeter', description: 'Application name/tag used in InfluxDB/Grafana')
        string(name: 'INFLUXDB_TOKEN_CREDENTIALS_ID', defaultValue: 'bb9bad5f8d27409a2ece8238093ba4aefbccb294f89789e8f8c5a3c8cce41bc6', description: 'Jenkins Secret Text credential ID storing InfluxDB token')
        booleanParam(name: 'FAIL_ON_INFLUXDB_ERROR', defaultValue: false, description: 'Fail build if InfluxDB credential/config is invalid; if false, run test without InfluxDB push')
    }

    options {
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 2, unit: 'HOURS')
    }

    environment {
        PTETESTDEMO_REPO   = 'https://github.com/praveenbe027/ptetestdemo.git'
        APACHEJMETER_REPO  = 'https://github.com/praveenbe027/ApacheJmeterTest.git'
        JMETER_VERSION     = "${params.JMETER_VERSION}"
        JMX_FILE           = "${params.JMX_FILE}"
        JMETER_HOME        = "${WORKSPACE}/apache-jmeter-${params.JMETER_VERSION}"
        JMETER_BIN         = "${WORKSPACE}/apache-jmeter-${params.JMETER_VERSION}/bin/jmeter"
        RESULTS_DIR        = "${WORKSPACE}/results"
        JTL_FILE           = "${WORKSPACE}/results/result.jtl"
        HTML_REPORT_DIR    = "${WORKSPACE}/html-report"
        JMETER_LOG         = "${WORKSPACE}/results/jmeter.log"
        INFLUXDB_URL          = "${params.INFLUXDB_URL}"
        INFLUXDB_ORG          = "${params.INFLUXDB_ORG}"
        INFLUXDB_BUCKET       = "${params.INFLUXDB_BUCKET}"
        INFLUXDB_APP          = "${params.INFLUXDB_APPLICATION}"
        INFLUXDB_SUMMARY_ONLY = "${params.INFLUXDB_SUMMARY_ONLY}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                echo 'Cleaning workspace before build...'
                script {
                    if (params.CLEAN_WORKSPACE) {
                        cleanWs(deleteDirs: true, disableDeferredWipeout: true, notFailBuild: true)
                    } else {
                        echo 'CLEAN_WORKSPACE=false, skipping workspace cleanup.'
                    }
                }
            }
        }

        stage('Checkout ptetestdemo') {
            steps {
                echo "Checking out ptetestdemo (branch: ${params.PTETESTDEMO_BRANCH})..."
                dir('ptetestdemo') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.PTETESTDEMO_BRANCH}"]],
                        userRemoteConfigs: [[url: env.PTETESTDEMO_REPO]]
                    ])
                }
            }
        }

        stage('Clone ApacheJmeterTest') {
            steps {
                echo "Cloning ApacheJmeterTest (branch: ${params.APACHEJMETER_BRANCH})..."
                dir('ApacheJmeterTest') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.APACHEJMETER_BRANCH}"]],
                        userRemoteConfigs: [[url: env.APACHEJMETER_REPO]]
                    ])
                }
                sh '''
                    set -eu
                    echo "ApacheJmeterTest contents:"
                    ls -la ApacheJmeterTest
                    if [ ! -f "ApacheJmeterTest/${JMX_FILE}" ]; then
                        echo "ERROR: JMX file '${JMX_FILE}' not found inside ApacheJmeterTest" >&2
                        exit 1
                    fi
                '''
            }
        }

        stage('Setup JMeter') {
            steps {
                echo "Setting up Apache JMeter ${params.JMETER_VERSION}..."
                sh '''
                    set -eu
                    JMETER_TGZ="apache-jmeter-${JMETER_VERSION}.tgz"
                    JMETER_URL="https://archive.apache.org/dist/jmeter/binaries/${JMETER_TGZ}"

                    if [ ! -d "${JMETER_HOME}" ]; then
                        echo "Downloading JMeter from ${JMETER_URL}"
                        curl -fSL --retry 3 -o "${JMETER_TGZ}" "${JMETER_URL}"
                        tar -xzf "${JMETER_TGZ}"
                        rm -f "${JMETER_TGZ}"
                    else
                        echo "JMeter already present at ${JMETER_HOME}"
                    fi

                    chmod +x "${JMETER_BIN}"
                    "${JMETER_BIN}" --version
                '''
            }
        }

        stage('Validate InfluxDB') {
            when {
                expression { return params.ENABLE_INFLUXDB }
            }
            steps {
                script {
                    try {
                        withCredentials([string(credentialsId: params.INFLUXDB_TOKEN_CREDENTIALS_ID, variable: 'INFLUXDB_TOKEN')]) {
                            sh '''
                                set +x
                                set -eu
                                echo "Validating InfluxDB connection and token access..."
                                curl -fsS \
                                    -H "Authorization: Token ${INFLUXDB_TOKEN}" \
                                    "${INFLUXDB_URL}/api/v2/buckets?org=${INFLUXDB_ORG}&limit=1" > /dev/null
                                echo "InfluxDB validation successful (token present and API reachable)."
                            '''
                        }
                    } catch (err) {
                        echo "InfluxDB validation failed: ${err.getMessage()}"
                        if (params.FAIL_ON_INFLUXDB_ERROR) {
                            error("Failing because FAIL_ON_INFLUXDB_ERROR=true and InfluxDB validation failed.")
                        }
                        echo 'Proceeding without InfluxDB push because FAIL_ON_INFLUXDB_ERROR=false.'
                    }
                }
            }
        }

        stage('Run JMeter Test') {
            steps {
                echo "Running JMeter test plan: ${params.JMX_FILE}"
                script {
                    if (params.ENABLE_INFLUXDB) {
                        try {
                            withCredentials([string(credentialsId: params.INFLUXDB_TOKEN_CREDENTIALS_ID, variable: 'INFLUXDB_TOKEN')]) {
                                sh '''
                                    set -eu
                                    mkdir -p "${RESULTS_DIR}"
                                    rm -f "${JTL_FILE}" "${JMETER_LOG}"

                                    INFLUX_WRITE_URL="${INFLUXDB_URL%/}/api/v2/write?org=${INFLUXDB_ORG}&bucket=${INFLUXDB_BUCKET}"
                                    echo "Pushing live metrics to: ${INFLUX_WRITE_URL}"

                                    "${JMETER_BIN}" -n \
                                        -t "ApacheJmeterTest/${JMX_FILE}" \
                                        -l "${JTL_FILE}" \
                                        -j "${JMETER_LOG}" \
                                        -Jjmeter.save.saveservice.output_format=csv \
                                        -Jbackend_metrics_window=10000 \
                                        -Jinfluxdb.url="${INFLUX_WRITE_URL}" \
                                        -Jinfluxdb.token="${INFLUXDB_TOKEN}" \
                                        -Jinfluxdb.application="${INFLUXDB_APP}" \
                                        -Jinfluxdb.measurement=jmeter \
                                        -Jinfluxdb.testTitle="${JOB_NAME}-${BUILD_NUMBER}" \
                                        -Jinfluxdb.samplersRegex=.* \
                                        -Jinfluxdb.percentiles="90;95;99" \
                                        -Jinfluxdb.summaryOnly="${INFLUXDB_SUMMARY_ONLY}"

                                    echo "InfluxDB integration enabled for Grafana time-series metrics."
                                    echo "JTL file generated:"
                                    ls -la "${JTL_FILE}"
                                '''
                            }
                        } catch (err) {
                            echo "InfluxDB integration skipped: ${err.getMessage()}"
                            if (params.FAIL_ON_INFLUXDB_ERROR) {
                                error("Failing because FAIL_ON_INFLUXDB_ERROR=true and InfluxDB setup failed.")
                            }
                            echo 'Continuing test execution without InfluxDB push.'
                            sh '''
                                set -eu
                                mkdir -p "${RESULTS_DIR}"
                                rm -f "${JTL_FILE}" "${JMETER_LOG}"

                                "${JMETER_BIN}" -n \
                                    -t "ApacheJmeterTest/${JMX_FILE}" \
                                    -l "${JTL_FILE}" \
                                    -j "${JMETER_LOG}" \
                                    -Jjmeter.save.saveservice.output_format=csv

                                echo "JTL file generated:"
                                ls -la "${JTL_FILE}"
                            '''
                        }
                    } else {
                        sh '''
                            set -eu
                            mkdir -p "${RESULTS_DIR}"
                            rm -f "${JTL_FILE}" "${JMETER_LOG}"

                            "${JMETER_BIN}" -n \
                                -t "ApacheJmeterTest/${JMX_FILE}" \
                                -l "${JTL_FILE}" \
                                -j "${JMETER_LOG}" \
                                -Jjmeter.save.saveservice.output_format=csv

                            echo "InfluxDB integration disabled for this run."
                            echo "JTL file generated:"
                            ls -la "${JTL_FILE}"
                        '''
                    }
                }
            }
        }

        stage('Generate HTML Report') {
            steps {
                echo 'Generating JMeter HTML dashboard report...'
                sh '''
                    set -eu
                    rm -rf "${HTML_REPORT_DIR}"
                    mkdir -p "${HTML_REPORT_DIR}"

                    "${JMETER_BIN}" -g "${JTL_FILE}" -o "${HTML_REPORT_DIR}"

                    echo "HTML report generated at: ${HTML_REPORT_DIR}"
                    ls -la "${HTML_REPORT_DIR}"
                '''
            }
        }

        stage('Publish HTML Report') {
            steps {
                echo 'Archiving and publishing HTML report...'
                archiveArtifacts artifacts: 'results/**/*, html-report/**/*', allowEmptyArchive: true, fingerprint: true

                publishHTML(target: [
                    allowMissing:          false,
                    alwaysLinkToLastBuild: true,
                    keepAll:               true,
                    reportDir:             'html-report',
                    reportFiles:           'index.html',
                    reportName:            'JMeter HTML Report',
                    reportTitles:          "Build #${env.BUILD_NUMBER}"
                ])
            }
        }
    }

    post {
        success {
            echo "Build #${env.BUILD_NUMBER} succeeded. HTML report published."
        }
        failure {
            echo "Build #${env.BUILD_NUMBER} failed. Check console log and archived results."
        }
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
