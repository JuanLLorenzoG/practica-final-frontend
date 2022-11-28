def version = ""
def pom = ""
pipeline {

    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: spring-boot-app
    image: juanllorenzogomis/jenkins-nodo-java-js-bootcamp:1.0
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-socket-volume
    securityContext:
      privileged: true
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - cat
    imagePullPolicy: IfNotPresent
    tty: true
  - name: newman
    image: postman/newman:latest
    command:
    - cat
    imagePullPolicy: IfNotPresent
    tty: true
  volumes:
  - name: docker-socket-volume
    hostPath:
      path: /var/run/docker.sock
      type: Socket
    command:
    - sleep
    args:
    - infinity
'''
            defaultContainer 'spring-boot-app'
        }
    }

	environment {
		DOCKER_HUB="jenkins_dockerhub"
		DOCKERHUB_CREDENTIALS=credentials("jenkins_dockerhub")
		DOCKER_IMAGE_NAME="juanllorenzogomis/practica-final-backend"
		NEXUS_VERSION = "nexus3"
		NEXUS_PROTOCOL = "http"
		NEXUS_URL = "192.168.49.3:8081"
		NEXUS_REPOSITORY = "bootcamp/"
		NEXUS_CREDENTIAL_ID = "jenkins_nexus"
	}

	stages {
		stage("Prepare environment"){
			steps {
				echo "version de java"
				sh 'java -version'
				echo "version de maven"
				sh 'mvn -version'
			}
		}

		stage("Compile"){
			steps {
				sh "mvn clean compile -DskipTests"
			}
		}

		stage("Unit Tests"){
			steps{
				sh "mvn test"
				junit "target/surefire-reports/*.xml"
			}
		}
		
		stage("JaCoCo Test"){
			steps{
				jacoco()
			}
		}

		stage("Quality Tests") {
			steps {
			        script {
					withSonarQubeEnv("sonarqube-server"){
						sh 'mvn clean verify sonar:sonar \
						-Dsonar.projectKey=practica-final-backend \
						-Dsonar.host.url=http://192.168.49.4:9000 \
						-Dsonar.login=sqp_0432952de91c6076b96cde584462c223a60d272e'
					}
				}
			}
		}

		stage("Package"){
			steps {
				sh 'mvn package -DskipTests'
			}
		}

		stage("Build & Push"){
			steps { 
				container('spring-boot-camp'){
					script {
						pom = readMavenPom(file: 'pom.xml')
						version = pom.version
					}
				}
				container('kaniko'){
					echo "Aqui se construye la imagen"
					script {
						withCredentials([usernamePassword(credentialsId: "jenkins_dockerhub", passwordVariable: "jenkins_dockerhubPassword", usernameVariable: "jenkins_dockerhubUser")]) {
							AUTH = sh(script: """echo -n "${env.jenkins_dockerhubUser}:${env.jenkins_dockerhubPassword}" | base64""", returnStdout: true).trim()
							command = """echo '{"auths": {"https://index.docker.io/v1/": {"auth": "${AUTH}"}}}' >> /kaniko/.docker/config.json"""
							sh("""
							set +x
							${command}
							set -x
							""")
							sh "/kaniko/executor --context `pwd` --destination ${DOCKER_IMAGE_NAME}:${version} --cleanup"
						}
					}
				}
			}
		}

		stage('Run Test Environment') {
			steps{
				script {
					if(fileExists("configuracion")){
						sh 'rm -r configuracion'
					}
				}
				sh 'git clone https://github.com/JuanLLorenzoG/kubernetes-helm-docker-config.git configuracion --branch test-implementation'
				sh 'kubectl apply -f configuracion/kubernetes-deployment/practica-final-backend/manifest.yml -n default --kubeconfig=configuracion/kubernetes-config/config'
				sleep 30 //seconds
			}
		}

	        stage ("Setup Jmeter") {
	            steps{
	                script {
	                    if(fileExists("jmeter-docker")){
		                    sh 'rm -r jmeter-docker'
	                    }

		                sh 'git clone https://github.com/JuanLLorenzoG/jmeter-docker.git jmeter-docker'

		                dir('jmeter-docker') {

		                    if(fileExists("apache-jmeter-5.5.tgz")){
		                        sh 'rm -r apache-jmeter-5.5.tgz'
		                    }

		                    sh 'wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.5.tgz'
		                    sh 'tar xvf apache-jmeter-5.5.tgz'
		                    sh 'cp plugins/*.jar apache-jmeter-5.5/lib/ext'
		                    sh 'mkdir test'
		                    sh 'mkdir apache-jmeter-5.5/test'
		                    sh 'cp *.jmx apache-jmeter-5.5/test/'
		                    sh 'chmod +775 ./build.sh && chmod +775 ./run.sh && chmod +775 ./entrypoint.sh'
		                    sh 'rm -r apache-jmeter-5.5.tgz'
		                    sh 'tar -czvf apache-jmeter-5.5.tgz apache-jmeter-5.5'
		                    sh './build.sh'
		                    sh 'rm -r apache-jmeter-5.5 && rm -r apache-jmeter-5.5.tgz'
		                    sh 'cp perform_test.jmx test'
		                }
	                }

	            }
	        }
        
	        stage ("Run Jmeter Performance Test") {
	            steps{
	                script {
	                    dir('jmeter-docker') {
	                        if(fileExists("apache-jmeter-5.5.tgz")){
	                            sh 'rm -r apache-jmeter-5.5.tgz'
	                        }
	                        sh './run.sh -n -t test/perform_test.jmx -l test/perform_test.jtl'
	                        sh 'pwd'
	                        sh 'docker cp jmeter:/home/jmeter/apache-jmeter-5.5/test/perform_test.jtl $(pwd)/test'
	                        perfReport 'test/perform_test.jtl'
	                    }
	                }
	            }
	        }
        
	        stage ("Generate Taurus Report") {
	            steps{
	                script {
	                    dir('jmeter-docker') {
	                        sh 'pip install bzt'
	                        sh 'export PATH=$PATH:/home/jenkins/.local/bin'

	                        BlazeMeterTest: {
	                            sh 'bzt test/perform_test.jtl -report'
	                        }
	                    }
	                }
	            }
	        }

		stage ("Run API Test") {
			steps{
				container('newman'){
					script {
						if(fileExists("practica-final-backend")){
							sh 'rm -r practica-final-backend'
						}
						sleep 20 // seconds
						container('spring-boot-app') {
							sh 'git clone https://github.com/JuanLLorenzoG/jmeter-docker.git practica-final-backend'
						}
						sh 'newman run jmeter-docker/bootcamp.postman_collection.json --reporters cli,junit --reporter-junit-export "newman/report.xml"'
						junit "newman/report.xml"
					}
				}

			}
		}

		stage("Publish to Nexus") {
			steps {
				script {
				// Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
				pom = readMavenPom file: "pom.xml"
				// Find built artifact under target folder
				filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
				// Print some info from the artifact found
				echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
				// Extract the path from the File found
				artifactPath = filesByGlob[0].path
				// Assign to a boolean response verifying If the artifact name exists
				artifactExists = fileExists artifactPath
					if(artifactExists) {
						echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}"
						version = "${pom.version}"

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
						classifier: "",
						file: artifactPath,
						type: pom.packaging],
						// Lets upload the pom.xml file for additional information for Transitive dependencies
						[artifactId: pom.artifactId,
						classifier: "",
						file: "pom.xml",
						type: "pom"]
						])
					} else {
						error "*** File: ${artifactPath}, could not be found"
					}
				}
			}
		}

	}

	post {
		always {
			cleanWs()
			sh "docker logout"
		}
	}

}
