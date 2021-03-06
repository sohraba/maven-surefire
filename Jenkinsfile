#!/usr/bin/env groovy

/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

properties(
    [
        buildDiscarder(logRotator(artifactDaysToKeepStr: env.BRANCH_NAME == 'master' ? '30' : '7',
                                  artifactNumToKeepStr: '10',
                                  daysToKeepStr: env.BRANCH_NAME == 'master' ? '30' : '7',
                                  numToKeepStr: '10')
        ),
        disableConcurrentBuilds()
    ]
)

final def oses = ['linux':'ubuntu', 'windows':'Windows && !windows-2012-3']
final def mavens = env.BRANCH_NAME == 'master' ? ['3.2.x', '3.3.x', '3.5.x'] : ['3.5.x']
final def jdks = [7, 8, 9, 10]

final def options = ['-e', '-V', '-B', '-nsu', '-P', 'run-its']
final def goals = ['clean', 'install', 'jacoco:report']
final Map stages = [:]

oses.eachWithIndex { osMapping, indexOfOs ->
    mavens.eachWithIndex { maven, indexOfMaven ->
        jdks.eachWithIndex { jdk, indexOfJdk ->
            def os = osMapping.key
            def label = osMapping.value
            final String jdkTestName = jenkinsEnv.jdkFromVersion(os, jdk.toString())
            final String jdkName = jenkinsEnv.jdkFromVersion(os, '8')
            final String mvnName = jenkinsEnv.mvnFromVersion(os, maven)
            final String stageKey = "${os}-jdk${jdk}-maven${maven}"

            def mavenOpts = '-server -XX:+UseG1GC -XX:+TieredCompilation -XX:TieredStopAtLevel=1 -XX:+UseNUMA \
-Xms64m -Djava.awt.headless=true'
            if (jdk > 7) {
                mavenOpts += ' -XX:+UseStringDeduplication'
            }
            mavenOpts += (os == 'linux' ? ' -Xmx1g' : ' -Xmx256m')

            if (label == null || jdkTestName == null || mvnName == null) {
                println "Skipping ${stageKey} as unsupported by Jenkins Environment."
                return;
            }

            println "${stageKey}  ==>  Label: ${label}, JDK: ${jdkTestName}, Maven: ${mvnName}."

            stages[stageKey] = {
                node(label) {
                    timestamps {
                        //https://github.com/jacoco/jacoco/issues/629
                        def boolean makeReports = os == 'linux' && indexOfMaven == mavens.size() - 1 && jdk == 9
                        def failsafeItPort = 8000 + 100 * indexOfMaven + 10 * indexOfJdk
                        def allOptions = options + ["-Dfailsafe-integration-test-port=${failsafeItPort}", "-Dfailsafe-integration-test-stop-port=${1 + failsafeItPort}"]
                        buildProcess(stageKey, jdkName, jdkTestName, mvnName, goals, allOptions, mavenOpts, makeReports)
                    }
                }
            }
        }
    }
}

timeout(time: 12, unit: 'HOURS') {
    try {
        parallel(stages)
        // JENKINS-34376 seems to make it hard to detect the aborted builds
    } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
        println "org.jenkinsci.plugins.workflow.steps.FlowInterruptedException: ${e}"
        // this ambiguous condition means a user probably aborted
        if (e.causes.size() == 0) {
            currentBuild.result = 'ABORTED'
        } else {
            currentBuild.result = 'FAILURE'
        }
        throw e
    } catch (hudson.AbortException e) {
        println "hudson.AbortException: ${e}"
        // this ambiguous condition means during a shell step, user probably aborted
        if (e.getMessage().contains('script returned exit code 143')) {
            currentBuild.result = 'ABORTED'
        } else {
            currentBuild.result = 'FAILURE'
        }
        throw e
    } catch (InterruptedException e) {
        println "InterruptedException: ${e}"
        currentBuild.result = 'ABORTED'
        throw e
    } catch (Throwable e) {
        println "Throwable: ${e}"
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        stage("notifications") {
            //jenkinsNotify()
        }
    }
}

def buildProcess(String stageKey, String jdkName, String jdkTestName, String mvnName, goals, options, mavenOpts, boolean makeReports) {
    cleanWs()
    try {
        def mvnLocalRepoDir = null
        if (isUnix()) {
            sh 'mkdir -p .m2'
            mvnLocalRepoDir = "${pwd()}/.m2"
        } else {
            bat 'mkdir .m2'
            mvnLocalRepoDir = "${pwd()}\\.m2"
        }

        println "Maven Local Repository = ${mvnLocalRepoDir}."
        assert mvnLocalRepoDir != null : 'Local Maven Repository is undefined.'

        stage("checkout ${stageKey}") {
            checkout scm
        }

        def properties = ["-Djacoco.skip=${!makeReports}", "\"-Dmaven.repo.local=${mvnLocalRepoDir}\""]
        println "Setting JDK for testing ${jdkName}"
        def cmd = ['mvn'] + goals + options + properties

        stage("build ${stageKey}") {
            if (isUnix()) {
                withEnv(["JAVA_HOME=${tool(jdkName)}",
                         "JAVA_HOME_IT=${tool(jdkTestName)}",
                         "MAVEN_OPTS=${mavenOpts}",
                         "PATH+MAVEN=${tool(mvnName)}/bin:${env.JAVA_HOME}/bin"
                ]) {
                    sh 'echo JAVA_HOME=$JAVA_HOME, JAVA_HOME_IT=$JAVA_HOME_IT'
                    def script = cmd + ['\"-Djdk.home=$JAVA_HOME_IT\"']
                    sh script.join(' ')
                }
            } else {
                withEnv(["JAVA_HOME=${tool(jdkName)}",
                         "JAVA_HOME_IT=${tool(jdkTestName)}",
                         "MAVEN_OPTS=${mavenOpts}",
                         "PATH+MAVEN=${tool(mvnName)}\\bin;${env.JAVA_HOME}\\bin"
                ]) {
                    bat 'echo JAVA_HOME=%JAVA_HOME%, JAVA_HOME_IT=%JAVA_HOME_IT%'
                    def script = cmd + ['\"-Djdk.home=%JAVA_HOME_IT%\"']
                    bat script.join(' ')
                }
            }
        }
    } finally {
        stage("reporting ${stageKey}") {
            if (makeReports) {
                openTasks(ignoreCase: true, canComputeNew: false, defaultEncoding: 'UTF-8', pattern: sourcesPatternCsv(),
                        high: tasksViolationHigh(), normal: tasksViolationNormal(), low: tasksViolationLow())

                jacoco(changeBuildStatus: false,
                        execPattern: '**/*.exec',
                        sourcePattern: sourcesPatternCsv(),
                        classPattern: classPatternCsv())

                junit(healthScaleFactor: 0.0,
                        allowEmptyResults: true,
                        keepLongStdio: true,
                        testResults: testReportsPatternCsv())

                if (currentBuild.result == 'UNSTABLE') {
                    currentBuild.result = 'FAILURE'
                }
            }

            if (fileExists('maven-failsafe-plugin/target/it')) {
                zip(zipFile: "maven-failsafe-plugin--${stageKey}.zip", dir: 'maven-failsafe-plugin/target/it', archive: true)
            }

            if (fileExists('surefire-its/target')) {
                zip(zipFile: "surefire-its--${stageKey}.zip", dir: 'surefire-its/target', archive: true)
            }

            archiveArtifacts(artifacts: "*--${stageKey}.zip", allowEmptyArchive: true, onlyIfSuccessful: false)
        }

        stage("cleanup ${stageKey}") {
            // clean up after ourselves to reduce disk space
            cleanWs()
        }
    }
}

@NonCPS
static def sourcesPatternCsv() {
    return '**/maven-failsafe-plugin/src/main/java,' +
            '**/maven-surefire-common/src/main/java,' +
            '**/maven-surefire-plugin/src/main/java,' +
            '**/maven-surefire-report-plugin/src/main/java,' +
            '**/surefire-api/src/main/java,' +
            '**/surefire-booter/src/main/java,' +
            '**/surefire-grouper/src/main/java,' +
            '**/surefire-its/src/main/java,' +
            '**/surefire-logger-api/src/main/java,' +
            '**/surefire-providers/**/src/main/java,' +
            '**/surefire-report-parser/src/main/java'
}

@NonCPS
static def classPatternCsv() {
    return '**/maven-failsafe-plugin/target/classes,' +
            '**/maven-surefire-common/target/classes,' +
            '**/maven-surefire-plugin/target/classes,' +
            '**/maven-surefire-report-plugin/target/classes,' +
            '**/surefire-api/target/classes,' +
            '**/surefire-booter/target/classes,' +
            '**/surefire-grouper/target/classes,' +
            '**/surefire-its/target/classes,' +
            '**/surefire-logger-api/target/classes,' +
            '**/surefire-providers/**/target/classes,' +
            '**/surefire-report-parser/target/classes'
}

@NonCPS
static def tasksViolationLow() {
    return '@SuppressWarnings'
}

@NonCPS
static def tasksViolationNormal() {
    return 'TODO,FIXME,@deprecated'
}

@NonCPS
static def tasksViolationHigh() {
    return 'finalize(),Locale.setDefault,TimeZone.setDefault,\
System.out,System.err,System.setOut,System.setErr,System.setIn,System.exit,System.gc,System.runFinalization,System.load'
}

@NonCPS
static def testReportsPatternCsv() {
    return '**/maven-failsafe-plugin/target/surefire-reports/*.xml,' +
            '**/maven-surefire-common/target/surefire-reports/*.xml,' +
            '**/maven-surefire-plugin/target/surefire-reports/*.xml,' +
            '**/maven-surefire-report-plugin/target/surefire-reports/*.xml,' +
            '**/surefire-api/target/surefire-reports/*.xml,' +
            '**/surefire-booter/target/surefire-reports/*.xml,' +
            '**/surefire-grouper/target/surefire-reports/*.xml,' +
            '**/surefire-its/target/surefire-reports/*.xml,' +
            '**/surefire-logger-api/target/surefire-reports/*.xml,' +
            '**/surefire-providers/**/target/surefire-reports/*.xml,' +
            '**/surefire-report-parser/target/surefire-reports/*.xml,' +
            '**/surefire-its/target/failsafe-reports/*.xml'
}
