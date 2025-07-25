pipeline {
    agent {
        kubernetes {
            label 'cd-template'
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  name: ci-pod
spec:
  containers:
    - name: node
      image: node:latest
      command:
        - sleep
      args:
        - 99d

    - name: sonar-scanner-cli
      image: sonarsource/sonar-scanner-cli:latest
      command:
        - sleep
      args:
        - 99d

    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command:
        - sleep
      args:
        - 99d
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
    - name: git
      image: bitnami/git
      command:
        - sleep
      args:
        - 99d
    - name: alpine
      image: alpine
      command:
        - sh
      args:
        - -c
        - "while true; do sleep 86400; done"

  restartPolicy: Never
  
  volumes:
    - name: kaniko-secret
      secret:
        secretName: docker-cred
        items:
          - key: .dockerconfigjson
            path: config.json
            '''
        }
    }

    options { 
        disableConcurrentBuilds()
    }

    environment {
        ARTIFACTORY_SERVER = 'trial5qmcqv.jfrog.io'
        ARGOCD_SERVER = 'argocd.192.168.49.2.sslip.io'
        ARGOCD_APP_NAME_STG = 'hello-world-staging'
        ARGOCD_APP_NAME_PROD = 'hello-world-prod'
        ARGOCD_TOKEN = credentials('argo_token')

    }

    stages {
        stage('Read Version Number') {
            steps {
                script {
                    VERSION = sh(script: 'grep \'"version":\' ./backend/package.json | head -1 | awk -F: \'{ print $2 }\' | sed \'s/[", ]//g\'', returnStdout: true).trim()
                }
            }
        }
        stage('Unit testing') {
            steps {
                container('node') {
                    sh '''
                        cd ./backend/
                        npm install
                        npm run test
                    '''
                }
            }
        }
        stage('SonarQube analysis') {
            steps {
                container('sonar-scanner-cli') {
                    withSonarQubeEnv('sonarqube') {
                        sh """
                            sonar-scanner \
                            -Dsonar.projectKey=hello \
                            -Dsonar.sources=backend/,frontend/,database-init/ \
                            -Dsonar.exclusions=test-output/  \
                            -Dsonar.tests=unit-tests/ \
                            -Dsonar.host.url=${env.SONAR_HOST_URL} \
                            -Dsonar.login=${env.SONAR_AUTH_TOKEN} \
                            -Dsonar.javascript.lcov.reportPaths=./test-output/coverage/lcov.info
                        """
                    }
                }
            }
        }
        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build and Publish Docker Images') {
            stages {
                stage('Frontend') {
                    steps {
                        container(name: 'kaniko', shell: '/busybox/sh') {
                            sh """
                                #!/busybox/sh
                                /kaniko/executor --context `pwd`/frontend --dockerfile=Dockerfile --insecure --destination=${ARTIFACTORY_SERVER}/jfrog-docker-local/frontend:${VERSION}-${BUILD_NUMBER} --image-name-with-digest-file=frontend-image-file
                            """
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        container(name: 'kaniko', shell: '/busybox/sh') {
                            sh """
                                #!/busybox/sh
                                /kaniko/executor --context `pwd`/backend --dockerfile=Dockerfile --insecure --destination=${ARTIFACTORY_SERVER}/jfrog-docker-local/backend:${VERSION}-${BUILD_NUMBER} --image-name-with-digest-file=backend-image-file
                            """
                        }
                    }
                }
            }
        }
        stage('Publish build info') {
            steps {
                rtCreateDockerBuild (
                    serverId: 'jfrog',
                    sourceRepo: 'jfrog-docker-local',
                    kanikoImageFile: "frontend-image-file"
                )
                rtCreateDockerBuild (
                    serverId: 'jfrog',
                    sourceRepo: 'jfrog-docker-local',
                    kanikoImageFile: "backend-image-file"
                )
                rtPublishBuildInfo (
                    serverId: 'jfrog'
                )
            }
        }
        stage('Update Staging Helm Chart Configuration') {
            steps {
                container('git') {
                    withCredentials([usernamePassword(credentialsId: 'github-repo-jenkins', passwordVariable: 'password', usernameVariable: 'username')]) {
                        sh """
                            curl -L https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -o /usr/bin/yq &&\
                            chmod +x /usr/bin/yq
                            git clone https://$username:$password@github.com/$username/hello-world.git && cd hello-world
                            git config user.email "${env.GIT_COMMITTER_EMAIL}"
                            git config user.name "${env.GIT_COMMITTER_NAME}"
                            yq eval '.backend.image.tag = \"${VERSION}-${BUILD_NUMBER}\"' values-staging.yaml -i
                            yq eval '.frontend.image.tag = \"${VERSION}-${BUILD_NUMBER}\"' values-staging.yaml -i
                            yq eval '.backend.image.repository = "${ARTIFACTORY_SERVER}/jfrog-docker-local/backend"' values-staging.yaml -i
                            yq eval '.frontend.image.repository = "${ARTIFACTORY_SERVER}/jfrog-docker-local/frontend"' values-staging.yaml -i
                            yq eval '.version = \"${VERSION}\"' Chart.yaml -i
                            git add .
                            git commit -m "Updated values-staging.yaml and Chart.yaml with new configurations"
                            git push origin main
                        """
                    }
                }
            }
        }
        stage('Verify Staging Deployment Health') {
            steps {
                container('alpine') {
                    sh '''
                        apk add --no-cache jq curl
                        DEPLOYMENT_INFO=$(curl -s -H "Authorization: Bearer $ARGOCD_TOKEN" "$ARGOCD_SERVER/api/v1/applications/$ARGOCD_APP_NAME_STG")

                        SYNC_STATUS=$(echo $DEPLOYMENT_INFO | jq -r '.status.sync.status')
                        HEALTH_STATUS=$(echo $DEPLOYMENT_INFO | jq -r '.status.health.status')

                        echo "Sync Status: $SYNC_STATUS"
                        echo "Health Status: $HEALTH_STATUS"
                    '''
                }
            }
        }
        stage('Run Performance Testing') {
            steps {
                container('alpine') {
                    sh '''
                        curl -L https://github.com/grafana/k6/releases/download/v0.46.0/k6-v0.46.0-linux-amd64.tar.gz -o k6.tar.gz
                        tar zxf k6.tar.gz
                        mv k6-v0.46.0-linux-amd64/k6 /usr/local/bin/
                        chmod +x /usr/local/bin/k6
                        k6 run performance-tests/performance-test.js
                    '''
                }
            }
        }
        stage('Update Production Helm Chart Configuration') {
            steps {
                container('git') {
                    withCredentials([usernamePassword(credentialsId: 'github-repo-jenkins', passwordVariable: 'password', usernameVariable: 'username')]) {
                        sh """
                            cd hello-world
                            yq eval '.backend.image.tag = \"${VERSION}-${BUILD_NUMBER}\"' values-production.yaml -i
                            yq eval '.frontend.image.tag = \"${VERSION}-${BUILD_NUMBER}\"' values-production.yaml -i
                            yq eval '.backend.image.repository = "${ARTIFACTORY_SERVER}/jfrog-docker-local/backend"' values-production.yaml -i
                            yq eval '.frontend.image.repository = "${ARTIFACTORY_SERVER}/jfrog-docker-local/frontend"' values-production.yaml -i
                            git add .
                            git commit -m "Updated values-production.yaml and Chart.yaml with new configurations"
                            git push origin main
                        """
                    }
                }
            }
        }
        stage('Verify Production Deployment Health') {
            steps {
                container('alpine') {
                    sh '''
                        DEPLOYMENT_INFO=$(curl -s -H "Authorization: Bearer $ARGOCD_TOKEN" "$ARGOCD_SERVER/api/v1/applications/$ARGOCD_APP_NAME_PROD")

                        SYNC_STATUS=$(echo $DEPLOYMENT_INFO | jq -r '.status.sync.status')
                        HEALTH_STATUS=$(echo $DEPLOYMENT_INFO | jq -r '.status.health.status')

                        echo "Sync Status: $SYNC_STATUS"
                        echo "Health Status: $HEALTH_STATUS"
                    '''
                }
            }
        }
    }
    post {
        always {
          junit '**/test-output/unit-test-report/junit-test-results.xml'
        }
    }
}
