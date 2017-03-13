#!groovy
/*
    Function for executing remote commands via SSH
    
    file_path : The path to the ssh private, used to connect to the remote host.
    host : The ip address of the remote host.
    cmd : The command or commands that will be executed at the remote host.
*/
def run_remote_docker(String file_path, String host, String cmd) {
    try{
        sh 'ssh -i '+file_path+' '+host+' "'+cmd+'"'
    }catch(err){
        throw err
    }
}
node{
    // NOTE: "*" means there is something to change
    
    //Global configurations
    def IPServer = '192.168.204.83'    
    def masterNode = '192.168.204.82'   
    def docker0 = '172.30.79.1'
    def projectName // should be the same a artifact id    
    
    //Gitlab configuration
    def gitPort = '8080'
    def git_repo_url = 'http://192.168.204.83:8080/TestLoginGroup/TestLoginTest.git' // * CHANGE IT
    def gitGroup = 'TestLoginGroup' // * CHANGE IT
    def gitRepositoryName = 'TestLoginTest' // * CHANGE IT
		env.BRANCH_NAME = 'sprint-v1.0.0'
    def branchName = env.BRANCH_NAME.split('-')[0] // DO NOT CHANGE    
    
    //Nexus configuration
    def nexusHttpPort = '8083'
    def nexusHttpsPort = '8446'
    
    // Maven repository 
    def repositoryNexus = 'portalviews-maven-snapshot' // * CHANGE IT
    
    //Docker repository
    def dockerRepository = 'portalviews-image-snapshot' // * CHANGE IT
    def dockerRepositoryHttps = '8447' // * CHANGE IT

 
    
    switch (branchName){
        case "sprint":
            
            stage('Repository Branch "'+env.BRANCH_NAME+'" Check Out'){        
                checkout([$class: 'GitSCM', 
                    branches: [[name: '*/'+env.BRANCH_NAME]], 
                    doGenerateSubmoduleConfigurations: false, 
                    extensions: [[$class: 'CleanBeforeCheckout']], 
                    submoduleCfg: [], 
                    userRemoteConfigs: [[credentialsId: 'gitlab-usr-pwd', 
                                        url: git_repo_url]]
                ])
            }
            
             stage('Run SQL') {
                def fly = tool name: 'Flyway', type: 'sp.sd.flywayrunner.installation.FlywayInstallation'
	            sh 'cp *.sql /var/jenkins_home/tools/sp.sd.flywayrunner.installation.FlywayInstallation/Flyway/flyway-4.0.3/sql/'
                sh fly+'/flyway-4.0.3/flyway -user=oortez -password=Avianca.2017 -url=jdbc:postgresql://192.168.204.83:5432/LBE_DEV -placeholders.abc=def migrate'
            }
            
            stage('SonarQube analysis') {
                // requires SonarQube Scanner 2.8+
                def scannerHome = tool name: 'SonarQube', type: 'hudson.plugins.sonar.SonarRunnerInstallation';
                withSonarQubeEnv{
                  sh "${scannerHome}/bin/sonar-scanner -X"
                }
            }
            
            stage("Apply build with Maven 3"){
                withMaven(jdk: 'java8', maven: 'mvn') {
                    // Run the maven build
                    sh "mvn clean install"
                }
            }
            
            //Version variables            
            def pomFile = readMavenPom file: ''
            def artifactId = pomFile.getArtifactId()
            projectName = artifactId;
            def versionPom = pomFile.getVersion()
            def groupId = pomFile.getGroupId()
            
            def versionArray = versionPom.split('-')
            def versionComp = versionArray[0]+'.'+env.BUILD_ID
            def finalVersion = versionArray[0]+'.'+env.BUILD_ID
            def fileName = env.WORKSPACE+'/target/'+artifactId+'-'+versionPom+'.jar'
            def jarName = artifactId+'-'+versionPom+'.jar'
            def jarRepository = 'https://'+IPServer+':'+nexusHttpsPort+'/repository/'+repositoryNexus+'/'+groupId.replace('.','/')+'/'+artifactId+'/'+finalVersion+'/'+artifactId+'-'+finalVersion+'.jar'
            
            stage('Upload jar version '+ finalVersion+' to Nexus'){
                nexusArtifactUploader artifacts: 
                                    [[artifactId: artifactId, 
                                        //classifier: 'default', 
                                        file: fileName, 
                                        type: 'jar'
                                    ]], 
                                        credentialsId: 'Nexus-Credentials-Rene', 
                                        groupId: groupId, 
                                        nexusUrl: IPServer+':'+nexusHttpsPort, 
                                        nexusVersion: 'nexus3', 
                                        protocol: 'https', 
                                        repository: repositoryNexus, 
                                        version: finalVersion    
            }
            
            stage('Build Docker Image'){
                run_remote_docker(  '~/.ssh/login-private-key',
    		                        docker0,
                                    'docker build --build-arg FILE='+jarRepository+' -t '+IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':'+finalVersion +' http://'+ IPServer + ':'+gitPort+'/'+gitGroup+'/'+gitRepositoryName+'/raw/'+ env.BRANCH_NAME +'/Dockerfile')
        		    
        		run_remote_docker(  '~/.ssh/login-private-key',
        		                    docker0,
        		                    'docker tag '+ IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':'+finalVersion+' '+IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':latest')
            }
            
            stage('Push Docker Image to Nexus'){
                
                run_remote_docker(  '~/.ssh/login-private-key',
                    		        docker0,
                    		        'docker push '+IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':'+finalVersion)
    		        
                run_remote_docker(  '~/.ssh/login-private-key',
                    		        docker0,
                    		        'docker push '+IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':latest')
                run_remote_docker(  '~/.ssh/login-private-key',
                    		        docker0,
                    		        'docker rmi '+IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':'+finalVersion)
                
                run_remote_docker(  '~/.ssh/login-private-key',
        		                    docker0,
        		                    'docker rmi '+IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':latest')            
                
            }
            
            stage('Deploy to dev env'){
                    run_remote_docker(  '~/.ssh/login-private-key',
            		                    masterNode,
            		                    'wget http://'+IPServer+':'+gitPort+'/'+gitGroup+'/'+gitRepositoryName+'/raw/'+env.BRANCH_NAME+'/'+projectName+'-create.sh;chmod 700 '+projectName+'-create.sh')
                    run_remote_docker(  '~/.ssh/login-private-key',
            		                    masterNode,
            		                    'wget http://'+IPServer+':'+gitPort+'/'+gitGroup+'/'+gitRepositoryName+'/raw/'+env.BRANCH_NAME+'/'+projectName+'-script.sh;chmod 700 '+projectName+'-script.sh')
                    run_remote_docker(  '~/.ssh/login-private-key',
            		                    masterNode,
            		                    './'+projectName+'-script.sh '+IPServer+':'+dockerRepositoryHttps+'/'+dockerRepository+':'+finalVersion+' dev'+' 4; rm '+projectName+'-script.sh '+projectName+'-create.sh '+projectName+'-controller.yaml')
            }
            break
    }
}
