# 1. 시작 전에 kube 환경에 helm 이 설치되어야 함 

# 2. jenkins 를 설치

+ helm repo add & pull
```shell
helm repo add jenkins https://charts.jenkins.io
helm pull jenkins/jenkins
tar -xvf jenkins-4.1.13.tgz
cd jenkins  
```
<br/>

+ jenkins agent 가 curl 과 docker 를 사용하기 위해 agent 이미지를 수동으로 생성
```yaml
#Curl 과 docker 를 사용하기 위한 Dockerfile
FROM jenkins/inbound-agent:alpine

USER root

# Alpine seems to come with libcurl baked in, which is prone to mismatching
# with newer versions of curl. The solution is to upgrade libcurl.
RUN apk update && apk add -u libcurl curl
# Install Docker client
ARG DOCKER_VERSION=18.03.0-ce
ARG DOCKER_COMPOSE_VERSION=1.21.0
RUN curl -fsSL https://download.docker.com/linux/static/stable/`uname -m`/docker-$DOCKER_VERSION.tgz | tar --strip-components=1 -xz -C /usr/local/bin docker/docker
RUN curl -fsSL https://github.com/docker/compose/releases/download/$DOCKER_COMPOSE_VERSION/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose

RUN touch /debug-flag
USER jenkins
```
위의 도커 파일을 build 후에 자신의 docker repo 에 push 해서 사용하자
```shell
# docker push 가 되려면 docker login 이 되어야 사용이 가능함
docker push ehgur1104/cicdtest:latest ( 실제 사용이미지는 팀원의 이미지를 사용함 )
```
<br/>

+ 받아진 Helm Jenkins 의 Value 파일 수정 
```shell
ubuntu@ip-192-168-56-211:~/jenkins$ ls
CHANGELOG.md  Chart.yaml  README.md  VALUES_SUMMARY.md  templates values.yaml
```
```shell
# 비밀번호 변경
48   adminUser: "admin"
49   adminPassword: "admin"
# service 수정
127   # For minikube, set this to NodePort, elsewhere use LoadBalancer
128   # Use ClusterIP if your setup includes ingress controller
129   serviceType: LoadBalancer
# agent 이미지 설정 
591   image: "human537/inbound-agent"
592   tag: "v1"
# docker mount 설정 (사용시 각각의 node에 docker.sock 의 권한을 변경 or 그룹을 추가해야함)
630    - type: HostPath
631      hostPath: /var/run/docker.sock
632      mountPath: /var/run/docker.sock
```
<br/>

+ devops 네임스페이스를 만들고 방금 수정한 value 파일을 통해 helm jenkins 설치

![image](https://user-images.githubusercontent.com/43317693/185622380-6dc4f364-cc62-4f1f-a4b1-a9300062a05b.png)
<br/>

+ Jenkins가 수정 권한을 가지기 위해서 clusterrolebing 을 해준다 
```shell
# 우선적으로 seviceaccount 확인 
ubuntu@ip-192-168-56-211:~/jenkins$ kubectl get sa -n devops
NAME      SECRETS   AGE
default   1         31m
jenkins   1         28m

# cluster 권한중 굉장히 강력한 cluster-admin 권한을 줌 
ubuntu@ip-192-168-56-211:~/jenkins$ kubectl create clusterrolebinding jenkins --clusterrole=cluster-admin --serviceacc
ount=devops:jenkins
clusterrolebinding.rbac.authorization.k8s.io/jenkins created
```
  
# 3. 젠킨스 구성
+ LoadBalancer 로 설정한 External-IP 로 접속
```shell
ubuntu@ip-192-168-56-211:~/jenkins$ kubectl get svc -n devops
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
jenkins         LoadBalancer   172.20.171.94   a9a9f59f196ba48fd9bd9762c1597279-1685923537.ap-northeast-2.elb.amazonaws.com   80:30838/TCP   45m
jenkins-agent   ClusterIP      172.20.114.90   <none>                                                                         50000/TCP      45m
```
<br/>

+ Jenkins plugin 설치
![image](https://user-images.githubusercontent.com/43317693/185624126-8fa225c4-35b8-477a-98a5-dd9d7e2023c0.png)
docker , kube , git 의 plugin 을 설치한다.
<br/>

+ credentials 생성하기

<br/>

git token
![image](https://user-images.githubusercontent.com/43317693/185624692-8fdab49f-13a7-46a3-8991-c9ab0932da30.png)

<br>

docker token 
![image](https://user-images.githubusercontent.com/43317693/185624727-2799d29d-c200-4ba5-bbd2-d7c69dd68acc.png)

<br>

jenkins token 
![image](https://user-images.githubusercontent.com/43317693/185624808-c5dc04df-f6f0-4158-a8f5-fb7b6ebf1479.png)
토큰값을 복호화 시켜서 사용한다.
![image](https://user-images.githubusercontent.com/43317693/185625062-a1277203-9d19-4a31-863a-8577a7fff3d1.png)

<br>

생성된 credentials
![image](https://user-images.githubusercontent.com/43317693/185625082-18ef3b02-8a4b-4626-a26c-16b6e6b738bf.png)

# 4. Jenkins File Review
```yaml
# kubernetes 로 agent 를 생성 
def podTemplate = """
apiVersion: v1
kind: Pod
metadata:
labels:
  jenkins/jenkins-jenkins-agent: "true"
  jenkins/label: "jenkins-jenkins-agent"
spec:
# docker sock 을 사용하기 위해서 그룹 권한을 securityContext 형식으로 지정
  securityContext:
    fsGroup: 1950    # Group ID of docker group on k8s nodes.
  serviceAccountName: jenkins
  containers:
  - name: topgun
    image: human537/inbound-agent:v1
    command:
    - cat
    tty: true
 # 컨테이너간 작업을 공유하기 위해서, docker 사용을 위해서 volume mount 를 
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
    - mountPath: /var/run/docker.sock
      name: docker-sock
      readOnly: true
    workingDir: "/home/jenkins/agent"
  - name: golang
    image: golang:1.10
    command:
    - cat
    tty: true
    workingDir: "/home/jenkins/agent"
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
  nodeSelector:
    kubernetes.io/os: "linux"
  restartPolicy: "Never"
  volumes:
  - name: "workspace-volume"
    emptyDir:
      medium: ""    
  - name: docker-sock
    hostPath:
      path: "/var/run/docker.sock"
"""

pipeline {
    agent {
        kubernetes {
/*          label 'sample-app' */
          yaml podTemplate
          defaultContainer 'topgun'
        }
    }
    environment {
        DOCKER_IMAGE_NAME = "ehgur1104/frontend"
    }
    stages {
        stage('Build Docker Image') {
            when {
                branch 'canary'
            }
            steps {
                container('topgun') {
                    sh ''
                    script {
                        app = docker.build(DOCKER_IMAGE_NAME,"--network host ./src/frontend/")
                        app.inside {
                            sh 'echo Hello, World!123'
                        }
                    }
                    echo 'Running Build Docker Image'
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'canary'
            }
            steps {     
                container('topgun') {
                    script {
                        docker.withRegistry('https://registry.hub.docker.com', 'docker-id') {
                            app.push("${env.BUILD_NUMBER}")
                            app.push("latest")
                        }
                    }
                    echo 'Running Push Docker Image'
                }
            }
        }
        stage('Canary') {
            when {
                branch 'canary'
            }
           steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    container('topgun') {
                        sh 'curl -LO "https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl"'
                        sh 'chmod u+x ./kubectl'
                        sh """
                           ./kubectl patch deployment frontend -n canary -p \
                           '{"spec":{"template":{"spec":{"containers":[{"name":"server","image":"${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"}]}}}}'
                           """
                    }
                }
           }          
        }
        stage('DeployToProduction') {
            when {
                branch 'main'
            }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    container('topgun') {
                        sh 'curl -LO "https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl"'
                        sh 'chmod u+x ./kubectl'
                        sh """
                           ./kubectl patch deployment frontend -n product -p \
                           '{"spec":{"template":{"spec":{"containers":[{"name":"server","image":"${DOCKER_IMAGE_NAME}:latest"}]}}}}'
                           """
                    }
                }
            }
        }
    }
}
```

# 5. 파이프라인
![image](https://user-images.githubusercontent.com/43317693/185637835-a1e07ef6-7101-4435-8d69-6e35150aecca.png)
![image](https://user-images.githubusercontent.com/43317693/185637877-1c775fbe-2f3e-4496-9cc8-182209014416.png)




