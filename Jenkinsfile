
pipeline {
    agent none

    //see https://www.jenkins.io/doc/book/pipeline/syntax/#options
    options {
        /*
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        disableResume()
        timeout(time: 1, unit: 'HOURS')
        skipDefaultCheckout()
        skipStagesAfterUnstable()
         */
    }

    stages {

        stage('prepare-build') {
            steps {
                runSingleTask("npm config", 6)
                runSingleTask("yarn install", 75)
                runSingleTask("download locales", 2)
            }
        }

        stage('parallelize-tests-and-builds') {
            parallel {
                stage('tests-line') {
                    steps {
                        runSingleTask("nx print-affected", 5)
                        script {
                            parallel(
                                    "paralellized tests": {
                                        runMultipleTasksParallel("Test", 3, 1500)
                                    },
                                    "paralellized lints": {
                                        runMultipleTasksParallel("Lint", 3, 1500)
                                    }
                            )
                        }
                        runSingleTask("merge and upload reports", 30)
                    }
                } // end of tests-way stage
                stage('builds-line') {
                    steps {
                        // 1 of 3 : Build all parts
                        script {
                            parallel(
                                    "Angular-builds": {
                                        stage("nx print-affected all") { runSingleTask("nx print-affected --all", 5) }
                                        stage("angular-builds") { runMultipleTasksParallel("angular-build", 3, 1200) }
                                    },
                                    "SSR-builds": {
                                        stage("Dependencies Restore") { runSingleTask("dependencies restore", 8) }
                                        stage("Build SSR") { runSingleTask("build SSR", 60) }
                                        stage("Unit Tests") { runSingleTask("UT .net framework", 40) }
                                    }
                            )
                        }
                        // 2 of 3 : At this point, both angular and SSR are built
                        runSingleTask("Builds output aggregation", 10)
                        // 3 of 3 : Finalization jobs : End to end tests, SSR local packaging, cleaning, bundle analysis
                        // Most can be run in parallel (E2E/.net/analysis)
                        script {
                            parallel(
                                    "E2E tests": {
                                        runMultipleTasksParallel("E2E-Test", 3, 60)
                                    },
                                    "Bundle Analyzer": {
                                        runMultipleTasksParallel("bundle-analysis", 3, 20)
                                    },
                                    "Packaging and cleaning": {
                                        runMultipleTasksParallel("publish-clean-package", 3, 10)
                                    },
                            )
                        }
                   } // end of build-way steps
                } // end of builds-way stage
            } // end parallel
        } // EOF stage "parallelize-tests-and-builds"

        // Last Stage

        stage('push-to-repos') {
            steps {
                script {
                    parallel(
                            "Push-to-Octopus": { runSingleTask("Push to octopus", 30) },
                            "Push-to-Nexus": { runSingleTask("Push to nexus", 15) },
                    )
                }
            }
        }

    } // end of main "stages"

    post {
        always {
            echo "Post : Always"
        }
        success {
            echo "Post : Success"
        }
        failure {
            echo "Post : Failure"
        }
    }

}

void runSingleTask(String label, int durationSeconds = 1) {
    echo "=== Simulating Job ===> ${label}"
    sleep(durationSeconds)
}

void runMultipleTasksParallel(String labelPrefix = "ParaStep", int nbSteps, int maxDurationSeconds) {
    def stepsToRun = [:]
    1.upto(nbSteps) { i ->
        def secondsToSleep = 1 + new Random().nextInt(maxDurationSeconds)
        stepsToRun["${labelPrefix}-${i}"] = {
            stage("${labelPrefix}-${i}") {
                runSingleTask("Step number ${i} doing some stuff for ${secondsToSleep} seconds...", secondsToSleep)
            }
        }
    }
    parallel stepsToRun
}
