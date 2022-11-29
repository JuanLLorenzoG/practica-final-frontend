def packageJSON = ""
def packageJSONVersion = ""
pipeline {

    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: super-nodo
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
            defaultContainer 'super-nodo'
        }
    }

	environment {
		DOCKER_HUB="jenkins_dockerhub"
		DOCKERHUB_CREDENTIALS=credentials("jenkins_dockerhub")
		DOCKER_IMAGE_NAME="juanllorenzogomis/practica-final-frontend"
		NEXUS_VERSION = "nexus3"
		NEXUS_PROTOCOL = "http"
		NEXUS_URL = "192.168.49.3:8081"
		NEXUS_REPOSITORY = "bootcamp/"
		NEXUS_CREDENTIAL_ID = "jenkins_nexus"
	}

	stages {
		stage("Prepare environment"){
			steps {
				echo "version de node"
				sh 'node --version'
			}
		}

		stage("Build"){
			steps {
				script {
					// Install dependencies/home/jenkins/agent/workspace/Practica_Final_Frontend_develop/functional-e2e-test). Please verify you invoked Maven from the correct directory. -> [Help 
					sh 'npm install'
					// Build assets with eg. webpack
					sh 'npm run build'
				}
			}
		}

		stage("Quality Tests") {
			steps {
			        script {
					withSonarQubeEnv(credentialsId: "sonarqube-credentials-js", installationName: "sonarqube-server"){
						sh 'npm run sonar'
					}
				}
			}
		}

		stage("Build & Push"){
			steps {
				container('kaniko'){
					echo "Aqui se construye la imagen"
					script {
						container('super-nodo'){
							packageJSON = readJSON file: 'package.json'
							packageJSONVersion = packageJSON.version
							echo packageJSONVersion
						}
						withCredentials([usernamePassword(credentialsId: "jenkins_dockerhub", passwordVariable: "jenkins_dockerhubPassword", usernameVariable: "jenkins_dockerhubUser")]) {
							AUTH = sh(script: """echo -n "${env.jenkins_dockerhubUser}:${env.jenkins_dockerhubPassword}" | base64""", returnStdout: true).trim()
							command = """echo '{"auths": {"https://index.docker.io/v1/": {"auth": "${AUTH}"}}}' >> /kaniko/.docker/config.json"""
							sh("""
							set +x
							${command}
							set -x
							""")
							sh "/kaniko/executor --context `pwd` --destination ${DOCKER_IMAGE_NAME}:${packageJSONVersion} --cleanup"
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
				sh 'kubectl apply -f configuracion/kubernetes-deployment/practica-final-frontend/manifest.yml -n default --kubeconfig=configuracion/kubernetes-config/config'
				sleep 30 //seconds
			}
		}

	        stage ('Run functional test e2e') {
	            steps{
	                script {
	                    if(fileExists("selenium")){
		                    sh 'rm -r selenium'
	                    }
				sh 'git clone https://github.com/JuanLLorenzoG/functional-e2e-test selenium --branch test-implementation'
				echo "Aqui va Selenium"
				dir('selenium') {
					script {
						sh 'mvn clean verify -Dwebdriver.remote.url=https://{ngrokUrl}/wd/hub -Dwebdriver.remote.driver=chrome -Dchrome.switches="--no-sandbox,--ignore-certificate-errors,--homepage=about:blank,--no-first-run,--headless"'
					}
				}
	                }

	            }
	        }
	        
		stage('Generate Cucumber Report') {
			steps{
				dir('selenium') {
					sh 'mvn serenity:aggregate'
	
					publishHTML(target: [
						reportName: 'Serenity',
						reportDir:  'target/site/serenity',
						reportFiles: 'index.html',
						keepAll: true,
						alwaysLinkToLastBuild: true,
						allowMissing: false
					])
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
