@Library('jenkins-common-library')

//Instanciate Objects from Libs
def util = new libs.utils.Util()

// Parameters to be used on job
properties([
    parameters([
        string(
            name: 'OCP_API_URL',
            defaultValue: '',
            description: 'OCP Server URL'
        ),
        string(
            name: 'OCP_CRED_USR',
            defaultValue: 'kubeadmin',
            description: 'OCP login user'
        ),
        password(name: 'OCP_CRED_PSW', description: 'User password'),
        string(
            name: 'OCP_SAMPLE_ARCH',
            defaultValue: 'x86',
            description: 'Sample Arch, for P env, value should be p, for Z env, value should be z'
        ),
        string(
            name: 'TEST_CASE',
            defaultValue: '',
            description: 'test case name, e.g. T1, T2. See tests/test_cases.go, default empty value will run all test cases.'
        )
    ])
])

// If the value is empty, so it was triggered by Jenkins, and execution is not needed (only pipeline updates).
if (util.getWhoBuild() == "[]") {
    // Define the build name and informations about it
    currentBuild.displayName = "Not Applicable"
    currentBuild.description = "Triggered Job"

    echo "Nothing to do!"

} else if (OCP_API_URL == "" | OCP_CRED_USR == "" | OCP_CRED_PSW == ""){
      // Define the build name and informations about it
      currentBuild.displayName = "Not Applicable"
      currentBuild.description = "Need more info"

      echo "Need to inform obrigatory fields!"

} else {

    node('jaeger'){
        // Define the build name and informations about it
        currentBuild.description = util.htmlDescription(util.whoBuild(util.getWhoBuild()))

        try {
            // Workspace cleanup and git checkout
            gitSteps()
            stage("Login in Openshift"){
                // Will print the masked value of the KEY, replaced with ****
                wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[var: 'OCP_CRED_PSW', password: OCP_CRED_PSW]], varMaskRegexes: []]) {
                    sh "oc login ${params.OCP_API_URL} -u=${params.OCP_CRED_USR} -p=${OCP_CRED_PSW} --insecure-skip-tls-verify"
                }
            }
            stage("Start running all tests"){
                dir('tests') {
                wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[var: 'OCP_CRED_PSW', password: OCP_CRED_PSW]], varMaskRegexes: []]) {
                    def OUT = sh (
                    script: """
                        if [ -z "${params.TEST_CASE}" ]; 
                        then docker run \
                        --name maistra-test-tool \
                        -d \
                        --rm \
                        --pull always \
                        -e SAMPLEARCH='${params.OCP_SAMPLE_ARCH}' \
                        -e OCP_CRED_USR='${params.OCP_CRED_USR}' \
                        -e OCP_CRED_PSW='${OCP_CRED_PSW}' \
                        -e OCP_API_URL='${params.OCP_API_URL}' \
                        -e GODEBUG=x509ignoreCN=0 \
                        quay.io/maistra/maistra-test-tool:2.1;
                        else echo;
                        fi
                    """,
                    returnStdout: true
                    ).trim()
                    println OUT
                }
                }
            }
            stage("Start running a single test case"){
                dir('tests') {
                wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[var: 'OCP_CRED_PSW', password: OCP_CRED_PSW]], varMaskRegexes: []]) {
                    def OUT = sh (
                    script: """
                        if [ -z "${params.TEST_CASE}" ]; 
                        then echo;
                        else docker run \
                        --name maistra-test-tool \
                        -d \
                        --rm \
                        --pull always \
                        -e SAMPLEARCH='${params.OCP_SAMPLE_ARCH}' \
                        -e OCP_CRED_USR='${params.OCP_CRED_USR}' \
                        -e OCP_CRED_PSW='${OCP_CRED_PSW}' \
                        -e OCP_API_URL='${params.OCP_API_URL}' \
                        -e TEST_CASE='${params.TEST_CASE}' \
                        -e GODEBUG=x509ignoreCN=0 \
                        --entrypoint "../scripts/pipeline/run_one_test.sh" \
                        quay.io/maistra/maistra-test-tool:2.1;
                        fi
                    """,
                    returnStdout: true
                    ).trim()
                    println OUT
                }
                }
            }
            stage ("Check Testing Completed") {
                def OUT = sh (
                script: """
                set +ex
                docker logs maistra-test-tool | grep "#Testing Completed#"
                while [ \$? -ne 0 ]; do sleep 60; docker logs maistra-test-tool | grep "#Testing Completed#"
                done
                set -ex
                """,
                returnStdout: true
                ).trim()
                println OUT
            }
            stage ("Collect logs") {
                def OUT = sh (
                script: """
                docker cp maistra-test-tool:/opt/maistra-test-tool/tests/test.log .
                docker cp maistra-test-tool:/opt/maistra-test-tool/tests/results.xml .
                """,
                returnStdout: true
                ).trim()
                println OUT
            }
            stage ("Validate Results") {
                def OUT = sh (
                script: """
                if grep -Fxq "FAIL" ${WORKSPACE}/test.log;
                then exit 1;
                else echo "Test Run PASS";
                fi
                """,
                returnStdout: true
                ).trim()
                println OUT
            }
        } catch(e) {
            currentBuild.result = "FAILED"
            throw e
        } finally {
            archiveArtifacts artifacts: 'test.log,results.xml'

            stage("Notify Results"){
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {            
                    // Additional information about the build
                    if (util.getWhoBuild() == "[]") {
                        executedBy = "Jenkins Trigger"
                    } else {
                        executedBy = util.whoBuild(util.getWhoBuild())
                    }                        
                    def moreInfo = "- Executed by: *${executedBy}*"

                    // Slack message to who ran the job
                    slackMessage(currentBuild.result,moreInfo,currentBuild.displayName)
                }
            } 
        }
    }  
}
