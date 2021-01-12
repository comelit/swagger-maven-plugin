pipeline {
    agent {
        docker {
            image 'gcr.io/ccs-container-registry/java-gitlab-runner-image:0.0.5'
            label 'white-pegasus'
            registryUrl 'https://gcr.io/ccs-container-registry/'
            registryCredentialsId 'gcrIoCredentialsJenkins'
            args '-v $HOME/.m2:$HOME/.m2 \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  -e HOME=$HOME \
                  -e CCS_MVN_SETTINGS=$CCS_MVN_SETTINGS \
                  -e CCS_MVN_SETTINGS_SECURITY=$CCS_MVN_SETTINGS_SECURITY \
                  -e CCS_DOCKER_USERNAME=$CCS_DOCKER_USERNAME \
                  -e CCS_DOCKER_PASSWORD=$CCS_DOCKER_PASSWORD \
                  -e CCS_DOCKER_URI=$CCS_DOCKER_URI'
        }
    }

    stages {
        stage('Init') {
            steps {
                script {
                    env.TAG = sh(returnStdout: true, script: 'git tag --points-at HEAD').trim()
                }
                sh 'mkdir -p $WORKSPACE/__jenkins_build_tmp'
                sh 'echo $CCS_MVN_SETTINGS > $WORKSPACE/__jenkins_build_tmp/settings.xml'
                sh 'echo $CCS_MVN_SETTINGS_SECURITY > $WORKSPACE/__jenkins_build_tmp/settings-security.xml'
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn compile -s $WORKSPACE/__jenkins_build_tmp/settings.xml -Dsettings.security=$WORKSPACE/__jenkins_build_tmp/settings-security.xml \
                -Duser.home=$HOME -Dmaven.test.skip=true dependency:go-offline'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test -s $WORKSPACE/__jenkins_build_tmp/settings.xml -Dsettings.security=$WORKSPACE/__jenkins_build_tmp/settings-security.xml \
                -Duser.home=$HOME dependency:go-offline'
            }
        }

        stage('Deploy') {
            when {
               expression { env.TAG != null && env.TAG != ""}
            }
            steps {
                    sh 'mvn compile -s $WORKSPACE/__jenkins_build_tmp/settings.xml -Dsettings.security=$WORKSPACE/__jenkins_build_tmp/settings-security.xml \
                    -Duser.home=$HOME -Dmaven.test.skip=true dependency:go-offline'
                    sh 'mvn deploy -s $WORKSPACE/__jenkins_build_tmp/settings.xml \
                    -Dsettings.security=$WORKSPACE/__jenkins_build_tmp/settings-security.xml -Duser.home=$HOME -Dmaven.test.skip=true dependency:go-offline'
                }
        }

        stage('Inform Slack for success') {
            steps {
                script {
                    if (env.TAG != null && env.TAG != "") {
                        slackSend tokenCredentialId: "slack-cloud-token", channel: "#jenkins-cloud", color: "good", message: "`${env.JOB_NAME}` \
                        \n*${env.BRANCH_NAME}* - *${env.TAG}* job is completed successfully"
                    }
                    if (env.TAG == null || env.TAG == "") {
                         slackSend tokenCredentialId: "slack-cloud-token", channel: "#jenkins-cloud", color: "good", message: "`${env.JOB_NAME}` \
                         \n*${env.BRANCH_NAME}* job is completed successfully"
                    }
                }
            }
        }
    }

    post {
       failure {
            slackSend tokenCredentialId: "slack-cloud-token", channel: "#jenkins-cloud", color: "danger", message: "*${env.JOB_NAME}* \
            \n*${env.BRANCH_NAME}* job is failed"
       }
       unstable {
            slackSend tokenCredentialId: "slack-cloud-token", channel: "#jenkins-cloud", color: "danger", message: "*${env.JOB_NAME}* \
             \n*${env.BRANCH_NAME}* job is unstable. Unstable means test failure, code violation etc."
       }
    }
}