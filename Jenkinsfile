pipeline {
  agent any //{ docker {
     // image 'bhanu3333/test:4'
    //  args '--user root -v /var/run/docker.sock:/var/run/docker.sock' // mount Docker socket to access the host's Docker daemon
   // }
 // }
   tools {
        jdk 'jdk21'
        maven 'mvn'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
  stages {
    stage('Install trivy') {
            steps {
                sh 'apt update && apt install openjdk-17-jdk -y'
                sh 'wget https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.deb'
                sh 'dpkg -i trivy_0.18.3_Linux-64bit.deb'

            }
    }
   // stage('Checkout') {
     // steps {
      //  sh 'echo passed'
        //git branch: 'test', url: 'https://github.com/vuyyuru-bhanu/spring-petclinic'
    //  }
   // }
   stage('File System Scan') {
           steps {
              sh "trivy fs --format table -o trivy-fs-report.html ."
           }
       }

  

    stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=sPetclinc \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=sPetclinic '''
                }
            }
        }
        stage("quality gate"){
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
           }
        }
        stage ('Build war file'){
            steps{
                sh 'mvn package'
            }
        }

        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
  
       stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "bhanu3333/springpetclinic:${BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = credentials('docker')
      }
      steps {
        script {
            sh 'docker build -t ${DOCKER_IMAGE} .'
			sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE} "
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker") {
                dockerImage.push()
            }
        }
      }
    }
    stage('Update Deployment File') {
        environment {
            GIT_REPO_NAME = "jpetstore-6"
            GIT_USER_NAME = "vuyyuru-bhanu"
            DOCKER_IMAGE = "bhanu3333/springpetclinic:${BUILD_NUMBER}"
        }
        steps {
            withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
               sh '''
                    git config user.email "bhanu.xyz@gmail.com"
                    git config user.name "bhanu"
                    sed -i "s|image: .*|image: ${DOCKER_IMAGE}|" manifests/deployements.yml
                    git add manifests/deployements.yml
                    git commit -m "Update deployment image to version of springpetclinic to  ${BUILD_NUMBER}"
                    git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:test
                '''
            }
        }
    }
  }
}