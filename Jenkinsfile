def lookupLastSuccessfulBuildId(String parent_jobname) {
  echo "Fetching last successful build ID of parent job:" + parent_jobname
  def lastSuccessfulBuildId = sh(script: "curl -s http://thunder.next:8080/job/${parent_jobname}/lastSuccessfulBuild/buildNumber", returnStdout: true)
  echo "BUILD_ID: " + lastSuccessfulBuildId
  return lastSuccessfulBuildId
}


pipeline {
    agent any

    parameters {
		string(name: 'RERUN_FAILING_TESTS', defaultValue: '0', description: 'How many time should Maven try to rerun failing tests')
        // -Dhttps.protocols=TLSv1.2 needs to move to Harmonia?
        string(name: 'MAVEN_OPTS', defaultValue: '-Dhttps.protocols=TLSv1.2', description: 'Extra settings for Maven')
    }

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
                    // warning, GIT_BRANCH var alreads points to pipeline's branch
                    if ( env.GIT_REPOSITORY_BRANCH == null || "".equals("${env.GIT_REPOSITORY_BRANCH}") ) {
                      env.GIT_REPOSITORY_BRANCH = "master"
                    }
                    echo "GIT_REPOSITORY_BRANCH:[${env.GIT_REPOSITORY_BRANCH}]"
                }
                dir('workdir') {
                  git url: "${env.GIT_REPOSITORY_URL}",
                      branch: "${env.GIT_REPOSITORY_BRANCH}"
                }
                dir('hera') {
                  git 'https://github.com/jboss-set/hera.git'
                }

                dir('harmonia') {
                  git branch: 'mvn_debug', url: 'https://github.com/rpelisse/harmonia.git'
                }

                script {
                    env.BUILD_SCRIPT = "${env.WORKSPACE}/hera/build-wrapper.sh"
					if ( "${env.BUILD_COMMAND}".startsWith("testsuite") ) {
                        def parent_jobname = "${env.JOB_NAME}".replace("-testsuite","-build")
                        assert ! "".equals(parent_jobname)
                        def lastSuccessfulBuildId = lookupLastSuccessfulBuildId(parent_jobname)
                        assert lastSuccessfulBuildId.isNumber()
                        env.PARENT_JOB_VOLUME = "/home/jenkins/jobs/${parent_jobname}/builds/${lastSuccessfulBuildId}/archive"
                        echo "Parent job workspace volume: ${env.PARENT_JOB_VOLUME}"
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
			findText(textFinders: [
                textFinder(regexp: /.INFO. BUILD FAILURE/, alsoCheckConsoleOutput: true, buildResult: 'UNSTABLE', changeCondition: 'MATCH_FOUND'),
                textFinder(regexp: /.INFO. BUILD ERROR/, alsoCheckConsoleOutput: true, buildResult: 'FAILURE', changeCondition: 'MATCH_FOUND')
            ])
            script {
			    try {
        		    sh label: '', script: "${env.WORKSPACE}/hera/hera.sh stop"
    		    } catch (err) {
        		    echo "Error while deleting container: ${err}"
    		    }
            }
        }
    }
}
