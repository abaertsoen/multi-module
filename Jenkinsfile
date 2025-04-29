pipeline {
    agent any 

    tools {
        maven "Maven_auto"
        jdk 'JDK21'
    }

    environment {
        SONAR_TOKEN = credentials('sonar_token')
        MAILING_LIST = 'global.team.fake@bnpparibas.com'
        stash_name = 'generated_artefact'
    }

    stages {
        stage('Compile et tests') {
            steps {
                echo 'Unit test et packaging'
                sh 'mvn -Dmaven.test.failure.ignore=true clean package'
            }
            post {
                always {
                    // Publier toujours les résultats des tests
                    junit '**/target/surefire-reports/*.xml'
                }
                success {
                    // En cas de succès : Archiver les artefacts
                    archiveArtifacts 'application/**/*.jar'
                    stash includes: 'application/**/*.jar', name: '${stash_name}'
                }
                failure {
                    // En cas d’erreur : Envoyer un mail
                    mail bcc: '', body: 'GO TROUBLESHOOT IT', cc: '', from: '', replyTo: '', subject: '[ERROR] Jenkins pipeline failed', to: '${MAILING_LIST}'
                }
            }
             
        }
        stage('Analyse qualité et vulnérabilités') {
            parallel {
                stage('Vulnérabilités') {
                    steps {
                        echo 'Tests de Vulnérabilités OWASP'
                        sh 'mvn -DskipTests verify'
                    }
                    
                }
                 stage('Analyse Sonar') {
                     steps {
                        echo 'Analyse sonar'
                        sh 'mvn -Dsonar.token=${SONAR_TOKEN} clean integration-test sonar:sonar'
                     }
                    
                }
            }
            
        }
            
        
        stage('Déploiement intégration') {
            //when { branch 'master' }
            steps {
                echo "Déploiement intégration"
                input cancel: 'Annuler', message: 'Dans quel Data Center, voulez-vous déployer l’artefact ?', ok: 'Déployer', parameters: [choice(choices: ['Paris', 'Lille', 'Lyon'], name: 'target_dc')]
                sh 'mkdir -p /home/plb/formation/workspace/deployments/${target_dc}'
                dir('/home/plb/formation/workspace/deployments/${target_dc}') {
                    unstash '${stash_name}'
                }
 
                //sh 'cp ${stash_name} /home/plb/formation/workspace/deployments/${target_dc}'
            }
        }

        

     }
    
}

