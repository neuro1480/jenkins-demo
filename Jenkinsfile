pipeline {
    agent none
    environment {
        IMAGE_NAME = "ghcr.io/neuro1480/jenkins-demo"   // 改成你的用户名
        TAG = "${BUILD_NUMBER}"                                 // 或 ${GIT_COMMIT.take(7)}
    }
    stages {
        stage('Checkout') {
            agent { label 'any' }   // 或直接用 kubernetes agent
            steps {
                checkout scm
            }
        }

        stage('Build & Push with Kaniko') {
            agent {
                kubernetes {
                    yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ["/busybox/cat"]
    tty: true
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker/
  volumes:
  - name: docker-config
    emptyDir: {}
'''
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'ghcr-credentials', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    container('kaniko') {
                        sh '''
                        # 创建 docker config.json 用于 GHCR 认证
                        mkdir -p /kaniko/.docker
                        echo "{\"auths\":{\"ghcr.io\":{\"auth\":\"$(echo -n $USER:$PASS | base64)\"}}}" > /kaniko/.docker/config.json
                        
                        /kaniko/executor \
                          --context . \
                          --dockerfile Dockerfile \
                          --destination ${IMAGE_NAME}:${TAG} \
                          --destination ${IMAGE_NAME}:latest \
                          --cache=true \
                          --cleanup
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            agent {
                kubernetes {
                    yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["cat"]
    tty: true
'''
                }
            }
            steps {
                withKubeConfig([credentialsId: 'k8s-cluster-credentials', serverUrl: '', contextName: '', clusterName: '']) {  // 在 Jenkins Credentials 配置 kubeconfig
                    container('kubectl') {
                        sh '''
                        # 替换镜像 tag
                        sed -i "s|PLACEHOLDER|${TAG}|g" k8s/deployment.yaml
                        kubectl apply -f k8s/
                        kubectl rollout status deployment/nginx-deployment
                        echo "部署成功！访问地址可通过 kubectl get svc nginx-service 查看"
                        '''
                    }
                }
            }
        }
    }
}
