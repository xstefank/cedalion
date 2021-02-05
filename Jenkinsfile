pipeline {
    agent any

    environment {
        HERA_USERNAME = 'jenkins'
        HERA_HOSTNAME = 'thunder.next'
    }

    stages {
        stage('Prep') {
            steps {
                //cleanWs()
                script {
                    // assert if job task is build or testsuite
					env.BUILD_COMMAND = "no-task"
                    echo "JOB_NAME:[${env.JOB_NAME}]"
                    if ( env.JAVA_HOME == null ) {
                      env.JAVA_HOME = "/opt/oracle/java"
                    }
                    echo "JAVA_HOME:[${env.JAVA_HOME}]"
                    if ( "${env.JOB_NAME}".endsWith("-build") ) {
                      env.BUILD_COMMAND = "build"
                    } else {
  					if ( "${env.JOB_NAME}".endsWith("-testsuite") ) {
                        env.BUILD_COMMAND = "testsuite"
                      } else {
                         currentBuild.result = 'ABORTED'
                         error("Invalid JOB_NAME: ${env.JOB_NAME}, can't determine BUILD_COMMAND (missing -build or -testsuite- suffix). Abort.")
                      }
                    }
                    echo "BUILD_COMMAND:[${env.BUILD_COMMAND}]"
                    // warning, GIT_BRANCH alreads points to pipeline's branch
                    echo "GIT_REPOSITORY_BRANCH:[${env.GIT_REPOSITORY_BRANCH}]"
                }
                dir('workdir') {
                  git url: "${env.GIT_REPOSITORY_URL}",
                      branch: "${env.GIT_REPOSITORY_BRANCH}"
                  // workaround intepol. issue in git: module
                  //sh label: '', script: "git checkout ${env.GIT_BRANCH} -b ${env.GIT_BRANCH}-${env.BUILD_ID}"
                }
                dir('hera') {
                  git 'https://github.com/rpelisse/hera.git'
                }

                dir('harmonia') {
                  git branch: 'mvn_debug', url: 'https://github.com/rpelisse/harmonia.git'
                }

                script {
                    env.BUILD_SCRIPT = "${env.WORKSPACE}/hera/build-wrapper.sh"

                    env.JAVA_HOME = "${env.JAVA_HOME}"
                    env.MAVEN_HOME = "${env.MAVEN_HOME}"
                    env.MAVEN_SETTINGS_XML = "${env.MAVEN_HOME}/conf/settings.xml"
                    env.MAVEN_OPTS = "${env.MAVEN_OPTS}"
                    env.TEST_TO_RUN = "${env.TEST_TO_RUN}"
                    env.RERUN_FAILING_TESTS = "${env.RERUN_FAILING_TESTS}"
                    // tweaks for wfly build
                    if ( ! "".equals(env.MAVEN_OPTS) ) {
                        env.MAVEN_OPTS = "-Dhttps.protocols=TLSv1.2"
                    }
                    echo "HERE: ${env.BUILD_COMMAND}"
					if ( "${env.BUILD_COMMAND}".startsWith("testsuite") ) {
                        def parent_jobname = "${env.JOB_NAME}".replace("-testsuite","-build")
                        //assert parent_jobname.isNotNull or empty
                        echo "Fetching last successful build ID of parent job:" + parent_jobname
                        def lastSuccessfulBuildId = sh(script: "curl -s http://thunder.next:8080/job/${parent_jobname}/lastSuccessfulBuild/buildNumber", returnStdout: true)
                        assert lastSuccessfulBuildId.isNumber()
                        echo "BUILD_ID: " + lastSuccessfulBuildId
                        env.PARENT_JOB_VOLUME = "/home/jenkins/jobs/${parent_jobname}/builds/${lastSuccessfulBuildId}/archive"
                        // add hera to checkout that folder exists
                        echo "${env.PARENT_JOB_VOLUME}"
					}
                }
                echo "BUILD_COMMAND: ${env.BUILD_COMMAND}"
                // Start container
		        sh label: '', script: "${env.WORKSPACE}/hera/hera.sh run"
            }
        }
		stage ('Build') {
            when { expression { env.BUILD_COMMAND == 'build' } }
            steps {
//                git "${env.GIT_REPOSITORY_URL}" # this, delete hera & harmonia
                sh label: '', script: 'pwd'
                sh label: '', script: "${env.WORKSPACE}/hera/hera.sh job"
				archiveArtifacts artifacts: '**/*', fingerprint: true, followSymlinks: false, onlyIfSuccessful: true
                // TODO trigger testsuite
            }
        }
		stage ('Testsuite') {
            when { expression { env.BUILD_COMMAND == 'testsuite' } }
            steps {
              sh label: '', script: "${env.WORKSPACE}/hera/hera.sh job"
            }
        }
    }
    post {
        always {
			try {
        		sh label: '', script: "${env.WORKSPACE}/hera/hera.sh stop"
    		} catch (err) {
        		echo "Error while deleting container: ${err}"
    		}
        }
    }
//    publishers {
//       findText {
//            textFinders {
//              textFinder {
//                   regexp '.INFO. BUILD FAILURE'
//                   changeCondition 'MATCH_FOUND'
//               alsoCheckConsoleOutput true
//                   buildResult 'UNSTABLE'
//        }
//      }
//    }
//  }
}
