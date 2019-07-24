## 개요 

여러 클라우드의 Kubernetes 서비스에서 쉽게 도커 레지스트리를 사용하기 위해 별도의 private registry 구성



## Registry 서버 구성

서버정보

OS : ubuntu 18.04

DNS : example.koreacentral.cloudapp.azure.com

PORT: 5000



### Registry 이미지 가져오기

```
$ docker pull registry
```



### SSL 인증서 발급

Let's Encrypt를 이용하여 인증서 발급

참조

<https://letsencrypt.org/>

<https://kr.minibrary.com/353/>

<https://gadelkareem.com/2018/10/23/deploy-a-docker-registry-with-letsencrypt-certificates-on-ubuntu-18-04/>



#### letsencrypt 설치

```
sudo apt update -y && sudo apt install letsencrypt -y
```



#### SSL 인증서 생성(80포트 사용)

```
$ sudo letsencrypt certonly --standalone -d example.koreacentral.cloudapp.azure.com
 
 
....
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example.koreacentral.cloudapp.azure.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example.koreacentral.cloudapp.azure.com/privkey.pem
   Your cert will expire on 2019-10-21. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:
 
   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```



#### 정해진 위치에 crt, key 복사

```
$ cd /etc/letsencrypt/live/example.koreacentral.cloudapp.azure.com
$ cp privkey.pem domain.key && \
$ cat cert.pem chain.pem > domain.crt && \
$ chmod 664 domain.*
 
$ chown gs:gs domain.*
$ mkdir /home/gs/certs
$ cp domain.* /home/gs/certs
```



### Registry 구성

참고 : <https://docs.docker.com/registry/deploying/>

#### create a password

```
$ mkdir auth
$ docker run \
  --entrypoint htpasswd \
  registry -Bbn testuser testpassword  > auth/htpasswd
```

#### registry start

```
$sudo docker run -d \
 --name local-registry \
 --restart=always \
 -p 5000:5000 \\
 -v /home/gs/certs:/certs \
 -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.crt \
 -e REGISTRY_HTTP_TLS_KEY=/certs/server.key \
 -v /home/gs/auth:/auth \
 -e "REGISTRY_AUTH=htpasswd" \
 -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
 -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
 registry
```

#### docker login

```
$ docker login example.koreacentral.cloudapp.azure.com:5000
```



## Client 설정

#### docker login

```
$ docker login example.koreacentral.cloudapp.azure.com:5000
Username: testuser
Password:
....
 
Login Succeeded
```



#### docker pull test

```
$ docker pull example.koreacentral.cloudapp.azure.com:5000/hello-world:gs
gs: Pulling from hello-world
Digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
Status: Downloaded newer image for example.koreacentral.cloudapp.azure.com:5000/hello-world:gs

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
$ docker images
REPOSITORY                                                       TAG                 IMAGE ID            CREATED             SIZE
example.koreacentral.cloudapp.azure.com:5000/hello-world   gs                  fce289e99eb9        6 months ago        1.84kB

```

