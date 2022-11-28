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
					def packageJSON = readJSON file: 'package.json'
					def packageJSONVersion = packageJSON.version
					echo packageJSONVersion
					// Install dependencies
					sh 'npm install'
					// Build assets with eg. webpack
					sh "VERSION=${packageJSONVersion} npm run build"
				}
			}
		}

		stage("Quality Tests") {
			steps {
			        script {
					withSonarQubeEnv("sonarqube-server"){
						sonar-scanner -Dsonar.projectKey=practica-final-frontend -Dsonar.sources=. -Dsonar.host.url=http://192.168.49.4:9000 -Dsonar.login=sqp_cf9a7aac6899e922f8bddfae3c9a722e689a70c0
					}
				}
			}
		}

		stage("Build & Push"){
			steps {
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
							sh "/kaniko/executor --context `pwd` --destination ${DOCKER_IMAGE_NAME}:${VERSION} --cleanup"
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

	        stage ("Selenium") {
	            steps{
	                script {
	                    if(fileExists("selenium")){
		                    sh 'rm -r selenium'
	                    }
				echo "Aqui va Selenium"
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
