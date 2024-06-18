pipeline{
    agent{
        label 'slave_1'
    }
    tools{
        jdk 'JAVA_HOME'
        maven 'MAVEN_HOME'
    }
    environment {
         SCANNER_HOME=tool 'sonar-server'
        DOCKERHUB_CREDENTIALS=credentials('DockerHubPass')
        // This can be nexus3 or nexus2
        NEXUS_VERSION = "nexus3"
        // This can be http or https
        NEXUS_PROTOCOL = "http"
        // Where your Nexus is running
        NEXUS_URL = "192.168.0.124:8081"
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = "repository-example"
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = "nexus-credentials"
      }
    stages{
        stage('SCM checkout'){
            steps{
                sh 'echo cloning the repo into the slave machine'
                git branch: 'main', url: 'https://github.com/re24ddy/Sundar-Anna.git'
            }
        }
        stage('compile source code'){
            steps{
                
                sh '''echo compiling the source code
                mvn clean compile'''
            }
        }
        stage('test compile source code'){
            steps{
                sh '''echo testing the compiled source code using suitable unit testing framework
                mvn test''' 
            }
        }
        stage('build project'){
            steps{
                sh '''echo cleaing the maven projct by deleting any existing target directory and build the project
                mvn clean package
                ls -al target'''
            }
        }
        
        stage('scan file system'){
            steps{
                sh '''echo scanning the files in the cloned git repository using trivy
                trivy fs --format table -o trivy-fs-report.html .'''
            }
        }
        stage("Sonarqube Analysis"){
             steps{
                 withSonarQubeEnv('sonar-server') {
                     sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BOARD_GAME \
                     -Dsonar.projectKey=BOARD_GAME \
                     -Dsonar.exclusions=**/*.java
                     '''
                 }
             }
         }
        stage("Quality Gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
         stage("publish to nexus") {
            steps {
                script {
                    // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
                    pom = readMavenPom file: "pom.xml";
                    // Find built artifact under target folder
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    // Print some info from the artifact found
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    // Extract the path from the File found
                    artifactPath = filesByGlob[0].path;
                    // Assign to a boolean response verifying If the artifact name exists
                    artifactExists = fileExists artifactPath;

                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                // Artifact generated such as .jar, .ear and .war files.
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],

                                // Lets upload the pom.xml file for additional information for Transitive dependencies
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );

                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }

        stage('Jfrog Artifact Upload') {
            steps {
              rtUpload (
                serverId: 'jfrog-server',
                spec: '''{
                      "files": [
                        {
                          "pattern": "*.jar",
                           "target": "local-snapshots"
                        }
                    ]
                }'''
              )
          }
        }

         stage('OWASP DP SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'owasp-dp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage('Build Docker Image') {
            steps {
                script{
                    sh 'docker build -t harish117/board_game_app .'
                }
            }
        }
        stage('Containerize And Test') {
            steps {
                script{
                    sh 'docker run -d --name board-game-app harish117/board_game_app && sleep 10 && docker stop board-game-app'
                }
            }
        }
        stage('Push Image To Dockerhub') {
            steps {
                script{
                    withCredentials([string(credentialsId: 'DockerHubPass', variable: 'DockerHubPass')]) {
                    sh 'docker login -u harish117 --password-stdin ${DockerHubPass}' }
                    sh 'docker push harish117/board_game_app:latest'
                }
            }
        }    
         stage("TRIVY Image Scan"){
            steps{
                sh "trivy image harish117/board_game_app:latest > trivyimage.txt" 
            }
        }
        // stage('Deploy to Kubernetes'){
        //     steps{
        //         script{
                    
        //                 withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
        //                         sh 'kubectl apply -f deployment-service.yaml'
                               
        //                         sh 'kubectl get svc'
        //                         sh 'kubectl get all'
        //                 }  
                    
        //         }
        //     }
        // }
       
      
        
    }
}
