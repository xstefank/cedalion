pipeline {
    agent any

    environment {
        HERO_HOME = './hera/'
        HERA_SSH_KEY = '/var/jenkins_home/jenkins.rsa'
        HERA_USERNAME = 'jenkins'
        HERA_HOSTNAME = 'thunder.next'
        MAVEN_HOME = '/opt/tools/apache-maven-3.6.3'
        JAVA_HOME = '/usr/lib/jvm/java-1.8.0-openjdk'
    }

    stages {
        stage('Build') {
            steps {
            	cleanWs()
                git "${env.GIT_REPOSITORY_URL}"
                dir('hera') {
                  git 'https://github.com/rpelisse/hera.git'
                }

                dir('harmonia') {
                  git 'https://github.com/jboss-set/harmonia.git'
                }

                // Start container
		        sh label: '', script: "./hera/hera.sh run > ${WORKSPACE}/cid.txt"
                script {
                    env.CID=readFile("${env.WORKSPACE}/cid.txt")
                    env.BUILD_SCRIPT = "${env.WORKSPACE}/hera/build-wrapper.sh"
                    env.HERA_HOME = "${env.WORKSPACE}/hera/"
                    env.MAVEN_SETTINGS_XML = "${env.MAVEN_HOME}/conf/settings.xml"
                    env.MAVEN_OPTS = "-Dhttps.protocols=TLSv1.2 -DskipTests=true"
                }
                // Run the build script on container
                sh label: '', returnStatus: true, script: './hera/hera.sh build'
        		// Stop the container
                sh label: '', script: "./hera/hera.sh stop"
            }
        }
    }
}
