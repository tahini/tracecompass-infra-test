pipeline {
	agent {
		kubernetes {
			label 'tracecompass-build'
			defaultContainer 'environment'
			yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: environment
    image: delislesim/eclipse-tracecompass-build-env:16.04
    imagePullPolicy: Always
    tty: true
    command: [ "/bin/sh" ]
    args: ["-c", "/home/tracecompass/.vnc/xstartup.sh && cat"]
    resources:
      requests:
        memory: "2.6Gi"
        cpu: "1.3"
      limits:
        memory: "2.6Gi"
        cpu: "1.3"
    volumeMounts:
    - name: settings-xml
      mountPath: /home/jenkins/.m2/settings.xml
      subPath: settings.xml
      readOnly: true
    - name: m2-repo
      mountPath: /home/jenkins/.m2/repository
    - name: tools
      mountPath: /opt/tools
  - name: jnlp
    image: 'eclipsecbi/jenkins-jnlp-agent'
    volumeMounts:
    - mountPath: /home/jenkins/.ssh
      name: volume-known-hosts
  volumes:
  - name: volume-known-hosts
    configMap:
      name: known-hosts
  - name: settings-xml
    configMap: 
      name: m2-dir
      items:
      - key: settings.xml
        path: settings.xml
  - name: m2-repo
    emptyDir: {}
  - name: tools
    persistentVolumeClaim:
      claimName: tools-claim-jiro-tracecompass
"""
		}
	}
	options {
        timestamps()
	    timeout(time: 4, unit: 'HOURS')
		// buildDiscarder(logRotator(numToKeepStr:'10'))
	}
	tools {
        maven 'apache-maven-latest'
        jdk 'oracle-jdk8-latest'
    }
	environment {
	    MAVEN_OPTS="-Xms768m -Xmx2048m"
	}
	stages {
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '$GERRIT_BRANCH_NAME']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'BuildChooserSetting', buildChooser: [$class: 'GerritTriggerBuildChooser']]], submoduleCfg: [], userRemoteConfigs: [[refspec: '$GERRIT_REFSPEC', url: '$GERRIT_REPOSITORY_URL']]])
            }
        }
        stage('Legacy') {
            when {
                expression { return params.LEGACY }
            }
            steps {
                sh 'cp -f ${WORKSPACE}/rcp/org.eclipse.tracecompass.rcp.product/legacy/tracing.product ${WORKSPACE}/rcp/org.eclipse.tracecompass.rcp.product/'
            }
        }
		stage('Build') {
			steps {
                // checkout([$class: 'GitSCM', branches: [[name: '$GERRIT_BRANCH_NAME']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'BuildChooserSetting', buildChooser: [$class: 'GerritTriggerBuildChooser']]], submoduleCfg: [], userRemoteConfigs: [[refspec: '$GERRIT_REFSPEC', url: '$GERRIT_REPOSITORY_URL']]])
				// git branch: 'master', url: 'git://git.eclipse.org/gitroot/tracecompass/org.eclipse.tracecompass'
                // sh 'mvn --version'
                // sh 'java -version'
                // sh 'echo $HOME'
                // sh 'mvn clean install -Pctf-grammar -Pbuild-rcp -Dmaven.test.error.ignore=true -Dmaven.test.failure.ignore=true -DskipTests -Dmaven.repo.local=/home/jenkins/.m2/repository --settings /home/jenkins/.m2/settings.xml'
                sh 'mvn clean install -Pctf-grammar -Pbuild-rcp -Dmaven.repo.local=/home/jenkins/.m2/repository --settings /home/jenkins/.m2/settings.xml ${MAVEN_ARGS}'
			}
			post {
				always {
					archiveArtifacts artifacts: '$ARCHIVE_ARTIFACTS', fingerprint: false
                    junit '**/target/surefire-reports/*.xml'
				}
			}
		}
		stage('Deploy') {
            when {
                expression { return params.DEPLOY }
            }
			steps {
				container('jnlp') {
					sshagent (['projects-storage.eclipse.org-bot-ssh']) {
						sh 'ssh genie.tracecompass@projects-storage.eclipse.org mkdir -p ${RCP_DESTINATION}'
                        sh 'ssh genie.tracecompass@projects-storage.eclipse.org mkdir -p ${RCP_SITE_DESTINATION}'
                        sh 'ssh genie.tracecompass@projects-storage.eclipse.org mkdir -p ${SITE_DESTINATION}' 
                        sh 'ssh genie.tracecompass@projects-storage.eclipse.org rm -rf  ${RCP_DESTINATION}*'
                        sh 'ssh genie.tracecompass@projects-storage.eclipse.org rm -rf  ${RCP_SITE_DESTINATION}*'
                        sh 'ssh genie.tracecompass@projects-storage.eclipse.org rm -rf  ${SITE_DESTINATION}*'
                        sh 'scp -r ${RCP_PATH} genie.tracecompass@projects-storage.eclipse.org:${RCP_DESTINATION}'
                        sh 'scp -r ${RCP_SITE_PATH} genie.tracecompass@projects-storage.eclipse.org:${RCP_SITE_DESTINATION}'
                        sh 'scp -r ${SITE_PATH} genie.tracecompass@projects-storage.eclipse.org:${SITE_DESTINATION}'
					}
				}
			}
		}
	}
    post {
        failure {
            emailext subject: 'Build $BUILD_STATUS: $PROJECT_NAME #$BUILD_NUMBER!', 
            body: '''$CHANGES
------------------------------------------
Check console output at $BUILD_URL to view the results.''',
            recipientProviders: [culprits(), requestor()], 
            to: '${EMAIL_RECIPIENT}'
        }
        fixed {
            emailext subject: 'Build is back to normal: $PROJECT_NAME #$BUILD_NUMBER!', 
            body: '''Check console output at $BUILD_URL to view the results.''',
            recipientProviders: [culprits(), requestor()], 
            to: '${EMAIL_RECIPIENT}'
        }
    }
}