pipeline {
    agent none 

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')
    }

    environment {
        SONAR_TOKEN = credentials('sonar_token')
        MAILING_LIST = 'global.team.fake@bnpparibas.com'
    }

    stages {
        stage('Compile et tests') {
            agent {
                kubernetes {
                    cloud 'KubLocal'
                    inheritFrom 'jdk17-agent'
                }
            }

            steps {
                echo 'Unit test et packaging'
                sh './mvnw -Dmaven.test.failure.ignore=true clean package'
            }

            post {
                always {
                    // Publier toujours les résultats des tests
                    junit '**/target/surefire-reports/*.xml'
                }
                success {
                    // En cas de succès : Archiver les artefacts
                    archiveArtifacts 'application/**/*.jar'
                    archiveArtifacts 'dist/*.tar.gz'
                    dir('application/target') {
                        stash includes: '*.jar', name: 'generated_artefact'
                    }
                }
                unsuccessful {
                    // En cas d’erreur : Envoyer un mail
                    mail bcc: '', body: 'GO TROUBLESHOOT IT', cc: '', from: '', replyTo: '', subject: '[ERROR] Jenkins pipeline failed', to: '${MAILING_LIST}'
                }
            }
             
        }
     }
    
}