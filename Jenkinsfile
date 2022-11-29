pipeline {
    agent {
        kubernetes {
          yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: shell
    image: juliocvp/jenkins-nodo-nodejs-bootcamp:1.0
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-socket-volume
    securityContext:
      privileged: true
  - name: "kaniko"
    image: "gcr.io/kaniko-project/executor:debug"
    command:
    - "cat"
    imagePullPolicy: "IfNotPresent"
    tty: true
  - name: java
    image: juliocvp/jenkins-nodo-java-bootcamp:1.0
    command:
    - "cat"
    imagePullPolicy: "IfNotPresent"
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
          defaultContainer 'shell'
        }
    }
    stages {
        stage('Prepare environment') {
            steps {
                sh 'node --version'
            }
        }
        stage('Build') {
            steps {
                // sh 'npm install && npm run build'
                sh 'npm install'
                sh 'npm run build &'
                sleep 30
            }
        }
        stage("Quality Tests") {
            steps {
                withSonarQubeEnv(credentialsId: "sonarqube-credentials", installationName: "sonarqube-server"){
                    sh 'npm run sonar'
                }
            }
        }
        stage('Build & Push') {
            steps {
                script {
                    APP_IMAGE_NAME = "practica-final-frontend"
                    APP_IMAGE_TAG = "latest"

                    container("kaniko") {
                        withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_HUB_PASS', usernameVariable: 'DOCKER_HUB_USER')]) {
                            AUTH = sh(script: """echo -n "${DOCKER_HUB_USER}:${DOCKER_HUB_PASS}" | base64""", returnStdout: true).trim()
                            command = """echo '{"auths": {"https://index.docker.io/v1/": {"auth": "${AUTH}"}}}' >> /kaniko/.docker/config.json"""
                            sh("""
                                set +x
                                ${command}
                                set -x
                                """)
                            sh "/kaniko/executor --dockerfile `pwd`/Dockerfile --context `pwd` --destination ${DOCKER_HUB_USER}/${APP_IMAGE_NAME}:${APP_IMAGE_TAG} --cleanup"
                        }
                    }

                }
            }
        }
        stage('Run test environment') {
            steps {
                //"juliocvp/practica-final-frontend:latest"
                sh "git clone https://github.com/juliocvp/kubernetes-helm-docker-config.git configuracion --branch test-implementation"

                script {
                    def imagen = "juliocvp/practica-final-frontend:latest"
                    sh("sed -i.bak 's#IMAGE_NAME#${imagen}#' ./configuracion/kubernetes-deployments/practica-final-frontend/deployment.yaml")
                }

                sh "kubectl apply -f configuracion/kubernetes-deployments/practica-final-frontend/deployment.yaml --kubeconfig=configuracion/kubernetes-config/config"
            }
        }
        stage('Selenium') {
            steps {
                sleep 30
                container("java"){
                    //levantar chrome-standalone
                    sh "kubectl apply -f configuracion/kubernetes-deployments/standalone-chrome/manifest.yml --kubeconfig=configuracion/kubernetes-config/config"
                    sleep 30
                    
                    sh "git clone https://github.com/juliocvp/functional-e2e-test.git testSelenium --branch practica-final"
                    //"https://chrome-standalone.loca.lt/"
                    //Run test
                    dir("testSelenium") {
                        sh "pwd"
                        sh 'mvn clean verify -Dwebdriver.remote.url=https://chrome-standalone.loca.lt/ -Dwebdriver.remote.driver=chrome -Dchrome.switches="--no-sandbox,--ignore-certificate-errors,--homepage=about:blank,--no-first-run,--headless"'

                        sh 'mvn serenity:aggregate'

                        publishHTML(target: [
                            reportName : 'Serenity',
                            reportDir:   'target/site/serenity',
                            reportFiles: 'index.html',
                            keepAll:     true,
                            alwaysLinkToLastBuild: true,
                            allowMissing: false
                        ])
                    }
                }
            }
        }
    }
    post {
        always {
          sh 'docker logout'
        }
    }
}
