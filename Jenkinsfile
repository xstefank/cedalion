pipeline {
    agent any

    environment {
        HERA_DEBUG = true
        HERA_USERNAME = 'jenkins'
        HERA_HOSTNAME = 'thunder.next'
    }

    stages {
        stage('Prep') {
            steps {
                //cleanWs()
                git branch: "${env.GIT_BRANCH}", url: "${env.GIT_REPOSITORY_URL}"
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
                    if ( ! "".equals(env.MAVEN_OPTS) )
                        env.MAVEN_OPTS = "-Dhttps.protocols=TLSv1.2"
                    // assert if job task is build or testsuite
					env.BUILD_COMMAND = "no-task"
                    if ( "${env.JOB_NAME}".endsWith("-build") ) {
                        env.BUILD_COMMAND = "build"
                    }
					if ( "${env.JOB_NAME}".endsWith("-testsuite") ) {
						env.BUILD_COMMAND = "testsuite"
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
		        sh label: '', script: "${WORKSPACE}/hera/hera.sh run"
            }
        }
		stage ('Build') {
            when { expression { env.BUILD_COMMAND == 'build' } }
            steps {
//                git "${env.GIT_REPOSITORY_URL}" # this, delete hera & harmonia
                sh label: '', script: 'pwd'
                sh label: '', script: './hera/hera.sh job'
				archiveArtifacts artifacts: '**/*', fingerprint: true, followSymlinks: false, onlyIfSuccessful: true
                // TODO trigger testsuite
            }
        }
		stage ('Testsuite') {
            when { expression { env.BUILD_COMMAND == 'testsuite' } }
            steps {
                sh label: '', script: './hera/hera.sh job'
            }
        }
		stage ('No task provided') {
            when { expression { env.BUILD_COMMAND == "no-task" } }
            steps {
              echo 'No task'
              //echo "INVALID BUILD_COMMAND PROVIDED: ${BUILD_COMMAND}".
			}
		}
    }
    post {
        always {
            sh label: '', script: "./hera/hera.sh stop"
        }
    }
}
