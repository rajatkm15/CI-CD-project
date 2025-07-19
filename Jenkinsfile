pipeline {
	agent{ kubernetes {
    label 'pod-template'
	  yaml '''
apiVersion: v1
kind: Pod
metadata:
  name: pod-template
spec:
  restartPolicy: Always
  containers:
   - name: node
     image: node:alpine
     command: ["/bin/sh","-c"]
     args: ["sleep 99d"]

   - name: sonarqube
     image: sonarsource/sonar-scanner-cli:latest
     command: ["/bin/sh","-c"]
     args: ["sleep 99d"]

   - name: kaniko
     image: gcr.io/kaniko-project/executor:debug
     command: ["/bin/sh","-c"]
     args: ["sleep 99d"]
     volumeMounts:
       - name: kaniko-volume
         mountPath: /kaniko/.docker
  volumes:
    - name: kaniko-volume
      secret:
        secretName: docker-cred
'''
 }
}
  stages {
    stage ('Unit test') {
      steps {
        container ('node') {
          sh '''
            cd ./backend
            npm install
            npm run test
            '''
          }
        }
      }
    stage ('Static Code Analysis') {
      steps {
        container('sonarqube') {
          withSonarQubeEnv ('sonarqube') {
            sh '''
               sonar-scanner \
               -Dsonar.projectKey=hello \
               -Dsonar.sources=backend,frontend,database-init \
               -Dsonar.exclusions=**/test-output/** \
               -Dsonar.tests=unit-test/ \
               -Dsonar.host.url=$(env.SONAR_HOST_URL) \
               -Dsonar.login=$(env.SONAR_AUTH_TOKEN) \
               -Dsonar.javascript.lcov.reportPaths=./test-output/coverage/lcov.info
              '''
            }
          }
        }
      }
    stage ('QualityGate') {
      steps {
        timeout (time: 1, unit: 'HOURS') {
          waitForQualityGate abortPipeline: true
           }
        }
      }
    stage ('Build-Image') {
      parallel {
        stage ('frontend') {
          steps {
            container('kaniko') {
              sh '''
                #!/busybox/sh
                VERSION=$(grep 'version' ./backend/package.json | head -1 | awk -F: '{ print $2 }' | sed s/[,"]//g )

                /kaniko/executor \
                --insecure --skip-tls-verify \
                --dockerfile=./frontend/Dockerfile \
                --destination=jfrog.192.168.49.2.sslip.io/artifactory/docker-local/frontend:$VERSION-$BUILD_NUMBER \
                --image-name-with-digest-file=frontend-image
                '''
                 }
              }
          }
        stage ('Backend') {
          steps {
            container('kaniko') {
              sh '''
                #!/busybox/sh
                VERSION=$(grep 'version' ./backend/package.json | head -1 | awk -F: '{ print $2 }' | sed s/[,"]//g )

                /kaniko/executor \
                --insecure \
                --skip-tls-verify \
                --dockerfile=./backend/Dockerfile \
                --destination=jfrog.192.168.49.2.sslip.io/artifactory/docker-local/backend:$VERSION-$BUILD_NUMBER \
                --image-name-with-digest-file=backend-image
                '''
              }
            }
          }
        }
    }
    stage ('PublishBuildInfo') {
      steps {
        rtCreateDockerBuild (
          serverId: 'jfrog',
          sourceRepo: 'docker-local',
          kanikoImageFile: 'frontend-image'
          )

        rtCreateDockerBuild (
          serverId: 'jfrog',
          sourceRepo: 'docker-local',
          kanikoImageFile: 'backend-image'
          )
        rtPublishBuildInfo (
          serverId: 'jfrog'
          )
        }
     }
 }
 post {
   always {
      junit '**/test-output/unit-test/result.xml'
      }
   }
}
