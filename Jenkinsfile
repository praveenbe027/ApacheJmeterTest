pipeline {
    agent any

    parameters {
        string(name: 'JMX_FILE',         defaultValue: 'Thread Group.jmx',                description: 'JMX test plan filename inside ApacheJmeterTest repo')
        booleanParam(name: 'ENABLE_INFLUXDB', defaultValue: true,                          description: 'Push live JMeter metrics to InfluxDB for Grafana')
        string(name: 'INFLUXDB_URL',     defaultValue: 'http://host.docker.internal:8086', description: 'InfluxDB v2 base URL reachable from Jenkins')
        string(name: 'INFLUXDB_ORG',     defaultValue: 'jmetertest',                       description: 'InfluxDB organization')
        string(name: 'INFLUXDB_BUCKET',  defaultValue: 'telegraf',                         description: 'InfluxDB bucket')
        string(name: 'INFLUXDB_TOKEN',   defaultValue: 'bb9bad5f8d27409a2ece8238093ba4aefbccb294f89789e8f8c5a3c8cce41bc6', description: 'InfluxDB API token (used as default when not overridden manually)')
        string(name: 'EMAIL_RECIPIENTS', defaultValue: 'praveenbe027@gmail.com',           description: 'Comma-separated email addresses to receive the HTML report')
    }

    options {
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 1, unit: 'HOURS')
    }

    environment {
        JMETER_VERSION  = '5.6.3'
        JMETER_HOME     = "${WORKSPACE}/apache-jmeter-5.6.3"
        JMETER_BIN      = "${WORKSPACE}/apache-jmeter-5.6.3/bin/jmeter"
        JTL_FILE        = "${WORKSPACE}/results/result.jtl"
        JMETER_LOG      = "${WORKSPACE}/results/jmeter.log"
        REPORT_DIR      = "${WORKSPACE}/html-report"
        JMX_FILE        = "${params.JMX_FILE}"
        ENABLE_INFLUXDB = "${params.ENABLE_INFLUXDB}"
        INFLUXDB_URL    = "${params.INFLUXDB_URL}"
        INFLUXDB_ORG    = "${params.INFLUXDB_ORG}"
        INFLUXDB_BUCKET = "${params.INFLUXDB_BUCKET}"
        INFLUXDB_TOKEN  = "${params.INFLUXDB_TOKEN}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs(deleteDirs: true, notFailBuild: true)
            }
        }

        stage('Checkout ptetestdemo') {
            steps {
                dir('ptetestdemo') {
                    git url: 'https://github.com/praveenbe027/ptetestdemo.git', branch: 'main'
                }
            }
        }

        stage('Clone ApacheJmeterTest') {
            steps {
                dir('ApacheJmeterTest') {
                    git url: 'https://github.com/praveenbe027/ApacheJmeterTest.git', branch: 'main'
                }
                sh '''
                    set -eu
                    test -f "ApacheJmeterTest/${JMX_FILE}" || { echo "JMX file '${JMX_FILE}' not found" >&2; exit 1; }
                '''
            }
        }

        stage('Setup JMeter') {
            steps {
                sh '''
                    set -eu
                    if [ ! -d "${JMETER_HOME}" ]; then
                        curl -fSL --retry 3 -o jmeter.tgz \
                            "https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-${JMETER_VERSION}.tgz"
                        tar -xzf jmeter.tgz && rm -f jmeter.tgz
                    fi
                    "${JMETER_BIN}" --version
                '''
            }
        }

        stage('Run JMeter Test') {
            steps {
                sh '''
                    set -eu
                    mkdir -p "$(dirname "${JTL_FILE}")"
                    rm -f "${JTL_FILE}" "${JMETER_LOG}"

                    if [ "${ENABLE_INFLUXDB}" = "true" ] && [ -n "${INFLUXDB_TOKEN}" ]; then
                        WRITE_URL="${INFLUXDB_URL%/}/api/v2/write?org=${INFLUXDB_ORG}&bucket=${INFLUXDB_BUCKET}"
                        QUERY_URL="${INFLUXDB_URL%/}/api/v2/query?org=${INFLUXDB_ORG}"
                        echo "Pushing live metrics to: ${WRITE_URL}"

                        echo "----- Pre-flight: writing a probe point to InfluxDB -----"
                        set +e
                        PROBE_TS=$(date +%s)
                        PROBE_BODY="jmeter,application=ptetest-jmeter,probe=preflight,build=${BUILD_NUMBER} value=1 ${PROBE_TS}000000000"
                        HTTP=$(curl -sS -o /tmp/influx_pre.out -w "%{http_code}" -X POST \
                            -H "Authorization: Token ${INFLUXDB_TOKEN}" \
                            -H "Content-Type: text/plain; charset=utf-8" \
                            --data-binary "${PROBE_BODY}" \
                            "${WRITE_URL}&precision=ns")
                        echo "Probe write HTTP: ${HTTP}"
                        if [ "${HTTP}" != "204" ] && [ "${HTTP}" != "200" ]; then
                            echo "Probe write FAILED. Response body:" >&2
                            cat /tmp/influx_pre.out >&2 || true
                            echo "Backend Listener will not work either. Check URL/org/bucket/token/network." >&2
                        else
                            echo "Probe write OK -> network, org, bucket and token are valid from Jenkins."
                        fi
                        set -e

                        "${JMETER_BIN}" -n \
                            -t "ApacheJmeterTest/${JMX_FILE}" \
                            -l "${JTL_FILE}" \
                            -j "${JMETER_LOG}" \
                            -Jjmeter.save.saveservice.output_format=csv \
                            -Jbackend_metrics_window=10000 \
                            -Jinfluxdb.url="${WRITE_URL}" \
                            -Jinfluxdb.token="${INFLUXDB_TOKEN}" \
                            -Jinfluxdb.measurement=jmeter \
                            -Jinfluxdb.testTitle="${JOB_NAME}-${BUILD_NUMBER}" \
                            -Jinfluxdb.samplersRegex='.*' \
                            -Jinfluxdb.summaryOnly=false

                        echo "----- jmeter.log tail (last 100 lines) -----"
                        tail -n 100 "${JMETER_LOG}" || true

                        echo "----- jmeter.log Backend/Influx-related lines -----"
                        grep -Ei 'influx|backend|HttpMetricsSender|sendMeasurement|did not respond|connection refused|401|404|400' "${JMETER_LOG}" || echo "(no influx/backend related log entries found)"

                        echo "----- Post-run: counting jmeter points written in last 5 minutes -----"
                        FLUX_QUERY='from(bucket:"'"${INFLUXDB_BUCKET}"'") |> range(start: -5m) |> filter(fn:(r)=> r._measurement == "jmeter") |> count()'
                        set +e
                        curl -sS -X POST \
                            -H "Authorization: Token ${INFLUXDB_TOKEN}" \
                            -H "Content-Type: application/vnd.flux" \
                            -H "Accept: application/csv" \
                            --data "${FLUX_QUERY}" \
                            "${QUERY_URL}" || echo "(query request failed)"
                        set -e
                        echo "----- Done diagnostics -----"
                    else
                        echo "InfluxDB push disabled or token missing; running JMeter without backend overrides."
                        "${JMETER_BIN}" -n \
                            -t "ApacheJmeterTest/${JMX_FILE}" \
                            -l "${JTL_FILE}" \
                            -j "${JMETER_LOG}" \
                            -Jjmeter.save.saveservice.output_format=csv
                    fi
                '''
            }
        }

        stage('Generate HTML Report') {
            steps {
                sh '''
                    set -eu
                    rm -rf "${REPORT_DIR}"
                    mkdir -p "${REPORT_DIR}"
                    "${JMETER_BIN}" -g "${JTL_FILE}" -o "${REPORT_DIR}"
                '''
            }
        }

        stage('Publish HTML Report') {
            steps {
                archiveArtifacts artifacts: 'results/**, html-report/**', allowEmptyArchive: true, fingerprint: true
                publishHTML(target: [
                    allowMissing:          false,
                    alwaysLinkToLastBuild: true,
                    keepAll:               true,
                    reportDir:             'html-report',
                    reportFiles:           'index.html',
                    reportName:            'JMeter HTML Report'
                ])
            }
        }

        stage('Send Email Notification') {
            when { expression { return params.EMAIL_RECIPIENTS?.trim() } }
            steps {
                sh '''
                    set -eu
                    cd "${WORKSPACE}"
                    rm -f html-report.tar.gz
                    [ -d html-report ] && tar -czf html-report.tar.gz html-report || true
                '''
                script {
                    def status = currentBuild.currentResult ?: 'IN_PROGRESS'
                    def stats  = fileExists('html-report/statistics.json') ? readFile('html-report/statistics.json').take(8000) : ''
                    emailext(
                        to:                 params.EMAIL_RECIPIENTS,
                        subject:            "[Jenkins] ${env.JOB_NAME} #${env.BUILD_NUMBER} - JMeter Report (${status})",
                        mimeType:           'text/html',
                        attachmentsPattern: 'html-report.tar.gz, html-report/statistics.json, results/result.jtl',
                        body: """
                            <h2>JMeter Test Report</h2>
                            <p><b>Job:</b> ${env.JOB_NAME} #${env.BUILD_NUMBER} &mdash; <b>Status:</b> ${status}</p>
                            <p>
                              <a href="${env.BUILD_URL}">Build</a> |
                              <a href="${env.BUILD_URL}JMeter_20HTML_20Report/">HTML Report</a> |
                              <a href="${env.BUILD_URL}artifact/">Artifacts</a>
                            </p>
                            ${stats ? "<h3>statistics.json</h3><pre style='background:#f6f8fa;padding:10px;border:1px solid #ddd;'>${stats}</pre>" : ''}
                            <p style='color:#666;font-size:12px;'>Full report attached as <code>html-report.tar.gz</code>.</p>
                        """.stripIndent()
                    )
                }
            }
        }
    }

    post {
        always { echo "Pipeline finished with status: ${currentBuild.currentResult}" }
    }
}
