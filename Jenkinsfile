@Library('GlobalLib') _

def target_dc
def target_deployments

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
                docker {
                    args '-v $HOME/.m2:/root/.m2'
                    image 'openjdk:17-alpine'
                }
            }

            steps {
                echo 'Unit test et packaging'
                sh './mvnw -Dmaven.test.failure.ignore=true clean package'
                createTarGz sourceDir:"application/target", extensions:["jar", "xml"], outputDir:"dist"
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
        
        stage('Analyse qualité et vulnérabilités') {
            parallel {
                stage('Vulnérabilités') {
                    tools {
                        maven "Maven_auto"
                        jdk 'JDK21'
                    }
                    agent any
                    steps {
                        echo 'Tests de Vulnérabilités OWASP'
                        sh 'mvn -DskipTests verify'
                    }
                    
                }
                stage('Analyse Sonar') {
                    tools {
                        maven "Maven_auto"
                        jdk 'JDK21'
                    }
                    agent any
                     steps {
                        echo 'Analyse sonar'
                        sh 'mvn -Dsonar.token=${SONAR_TOKEN} clean integration-test sonar:sonar'
                        script {
                            checkSonarQualityGate()
                        }    
                     }
                }
            }
            
        }

        stage('Ansible integration'){
            tools {
                ansible 'ansible'
            }
            agent any
            steps {
                ansibleAdhoc(credentialsId: 'b81d130f-8cd3-49f5-9932-2d644f411b4c', inventory: '/home/plb/formation/workspace/ansible/inventory.list', hosts: 'slaves', module: 'shell', moduleArguments: 'df -Th')
               //ansiblePlaybook credentialsId: 'b81d130f-8cd3-49f5-9932-2d644f411b4c', disableHostKeyChecking: true, installation: 'ansible', inventory: '/home/plb/formation/workspace/ansible/inventory.list', playbook: '/home/plb/formation/workspace/ansible/run_script.yml', vaultTmpPath: ''
            }
        }  
        
        stage('Push to docker hub') {
            agent any
            steps {
                script {
                    def dockerImage = docker.build('firstdockerfile/multi-module', '.')
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub') {
                        dockerImage.push "${env.BRANCH_NAME}"
                    }
                }
            }
        }

        stage('Déploiement via configuration') {
            /*when {
                branch 'master'
                beforeInput true
                beforeAgent true
                beforeOptions true
            }*/
            agent any

            steps {
                script {
                    target_deployments = readJSON file: 'deployment_vars.json'

                    echo "Déploiement via config"
                    //echo "Deploying conf ${target_deployments}"
                
                    for(i in target_deployments["dataCenters"]) { 
                        println "Deploying to ${i}"
                        sh "mkdir -p ${target_deployments['integrationURL']}/autos/${i}"
                        dir("${target_deployments['integrationURL']}/${i}") {
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

     }
    
}

def checkSonarQualityGate(){
    // Get properties from report file to call SonarQube 
    def sonarReportProps = readProperties  file: 'target/sonar/report-task.txt'
    def sonarServerUrl = sonarReportProps['serverUrl']
    def ceTaskUrl = sonarReportProps['ceTaskUrl']
    def ceTask

    // Get task informations to get the status
    timeout(time: 4, unit: 'MINUTES') {
        waitUntil(initialRecurrencePeriod: 1000)  {
            withCredentials ([string(credentialsId: 'sonar_token', variable : 'token')]) {
                def response = sh(script: "curl -u ${token}: ${ceTaskUrl}", returnStdout: true).trim()
                ceTask = readJSON text: response
            }

            echo ceTask.toString()
              return "SUCCESS".equals(ceTask['task']['status'])
        }
    }

    // Get project analysis informations to check the status
    def ceTaskAnalysisId = ceTask['task']['analysisId']
    def qualitygate

    withCredentials ([string(credentialsId: 'sonar_token', variable : 'token')]) {
        def response = sh(script: "curl -u ${token}: ${sonarServerUrl}/api/qualitygates/project_status?analysisId=${ceTaskAnalysisId}", returnStdout: true).trim()
        qualitygate =  readJSON text: response
    }

    echo qualitygate.toString()
    if ("ERROR".equals(qualitygate['projectStatus']['status'])) {
        error "Quality Gate failure"
    }
}