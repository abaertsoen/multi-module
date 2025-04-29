def target_dc
def target_dcs

pipeline {
    agent none 

    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')
    }

    tools {
        maven "Maven_auto"
        jdk 'JDK21'
        ansible 'ansible'
    }

    environment {
        SONAR_TOKEN = credentials('sonar_token')
        MAILING_LIST = 'global.team.fake@bnpparibas.com'
    }

    stages {
        stage('Compile et tests') {
            agent any
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
        stage('Analyse qualité et vulnérabilités') {
            parallel {
                stage('Vulnérabilités') {
                    agent any
                    steps {
                        echo 'Tests de Vulnérabilités OWASP'
                        sh 'mvn -DskipTests verify'
                    }
                    
                }
                stage('Analyse Sonar') {
                    agent any
                     steps {
                        echo 'Analyse sonar'
                        sh 'mvn -Dsonar.token=${SONAR_TOKEN} clean integration-test sonar:sonar'
                     }
                    
                }
            }
            
        }

        stage('Ansible integration'){
            agent any
            steps {
                ansibleAdhoc(credentialsId: 'b81d130f-8cd3-49f5-9932-2d644f411b4c', inventory: '/home/plb/formation/workspace/ansible/inventory.list', hosts: 'slaves', module: 'shell', moduleArguments: 'df -Th')
               //ansiblePlaybook credentialsId: 'b81d130f-8cd3-49f5-9932-2d644f411b4c', disableHostKeyChecking: true, installation: 'ansible', inventory: '/home/plb/formation/workspace/ansible/inventory.list', playbook: '/home/plb/formation/workspace/ansible/run_script.yml', vaultTmpPath: ''
            }
        }  

        stage('Déploiement via configuration') {
            when {
                branch 'master'
                beforeInput true
                beforeAgent true
                beforeOptions true
            }
            agent any

            steps {
                target_deployments = readJSON file: 'deployment_vars.json', text: ''

                echo "Déploiement via config"
                echo "Deploying to ${target_dcs}"
                script {
                    for(i in target_deployments["dataCenters"]) { 
                        println "Deploying to ${i}"
                        sh 'mkdir -p ${target_deployments["integrationURL"]}/deployments/${i}'
                        dir('${target_deployments["integrationURL"]}/${i}') {
                            unstash 'generated_artefact'
                        }
                    }
                }    
                  
            }
        }
        
        stage('Déploiement validation') {
            when {
                branch 'master'
                beforeInput true
                beforeAgent true
                beforeOptions true
            }

            input {
                message 'Dans quel Data Center, voulez-vous déployer l’artefact ?'
                ok 'Déployer'
                submitterParameter 'target_dc_approver'
                parameters {
                    choice choices: ['Paris', 'Lille', 'Lyon'], name: 'TARGETDC'
                }
            }

            steps {
                script {
                    target_dc = ${env.TARGETDC}
                }
            }
        }

        stage('Déploiement intégration') {
            when {
                branch 'master'
                beforeInput true
                beforeAgent true
                beforeOptions true
            }
            agent any

            steps {
                echo "Déploiement intégration"
                echo "Deploying to ${target_dc}"
                sh "mkdir -p /home/plb/formation/workspace/deployments/${target_dc}"
                dir("/home/plb/formation/workspace/deployments/${target_dc}") {
                    unstash 'generated_artefact'
                }
            }
        }

        /* 
        stage('Déploiement via json file'){
            readJSON file: '/data', text: ''
        }
        */

     }
    
}

