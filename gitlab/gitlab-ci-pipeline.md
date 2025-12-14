## General Knowledge 
How to add a script in your pipeline:
before dive in:
- **`default:`** — defines default settings that apply to _all_ jobs in the pipeline (unless overridden in a specific job).
    
- **`tags:`** — tells GitLab which **runner** should pick up the job.  
    The runner must have the same tag configured (in your case: `docker-based-pipeline-zag`).
    
- **`image:`** — defines the **Docker image** to use for all jobs.  
    Here it’s pulling `alpine:latest` from your private GitLab container registry.  
    This means all jobs will run inside this lightweight Alpine Linux container.
.gitlab-ci.yml :
```
default:
  tags:
    - docker-based-pipeline-zag   # 👈 your runner’s tag
  image: 192.168.0.110:5050/cd/folder1/test1/temp1/alpine:latest
stages:
  - build
  - test
  - deploy
build-job:
  stage: build
  script:
    - echo "Building the project..."
    - echo "NOW THE SCRIPT WILL RUN!!!!!!!!!!!!!!!!!!!"
    #- chmod +x ./script.sh
    - pwd
    - ls -la
    - cat ./script.sh
    - dos2unix script.sh
    - chmod +x ./script.sh && ./script.sh
test-job:
  stage: test
  script:
    - echo "Running tests..."
    - echo "All tests passed!"
    - echo "All tests passed!"
    - cd /home
    - ls -la
    - cat /etc/hosts
deploy-job:
 # needs:
  #  - test-job
  stage: deploy
  script:
    - df -h
    - cd /tmp/
    - ls -la
deploy-job-second:
 # needs:
  #  - test-job
  stage: deploy
  script:
    - df -h
    - cd /usr/local
    - ls -la
```


Just run the specific job on specific branch:
```
test-job:
  stage: test
  only:
    - main
  script:
    - echo "Running tests..."
      ...
```


 run the specific job on dev branch:
```
test-job:
  stage: test
  only:
    - dev
  script:
    - echo "Running tests..."
      ...
```
git command for creation new branch:
```
git checkout -b dev
git add .
git commit -m "explain"
git push origin dev
```

Variables:
we have predefined variables and also you can define in your file:
```
variables:
   local_image_repository: 192.168.0.110:5050/cd/folder1/test1/temp1
   ...
     - echo " local image repository is $local_image_repository "
   
```
link for predefined variables:
https://docs.gitlab.com/ci/variables/predefined_variables/

What is `workflow:` in GitLab CI/CD?
The `workflow:` keyword defines **rules for the entire pipeline**, not just one job.  
It decides **whether the pipeline should run at all** based on conditions like:

- the branch or tag name
- the type of pipeline trigger (push, merge request, schedule, etc.)
- or even custom variables
```
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - when: never

```
Skip pipeline if only docs changed
```
workflow:
  rules:
    - changes:
        - README.md
        - docs/**        # all files inside the docs/ folder
        - "*.md"         # any other Markdown files
      when: never
    - when: always

```
Set two rules():
```
workflow:
  rules:
  
    # Don’t run the pipeline if only README.md changed
    - changes:
        - README.md
      when: never
  
    # Run only for the main branch
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always

    # Otherwise, never run

    - when: never
```

Using specidic image for a specific job :
```
test-job:
  image: 192.168.0.110:5050/cd/folder1/test1/temp1/docker:latest
  stage: test
```
in Gitlab container registry you must add the image in the project!
```
docker tag docker.arvancloud.ir/docker:latest 192.168.0.110:5050/cd/folder1/test1/temp1/docker:latest

docker push 192.168.0.110:5050/cd/folder1/test1/temp1/docker:latest
```



## Project Ansible and CICID
**The project for creation ansible image custom has created.**
This project creates the proper image for ansible playbook and pushes the image to the registry of it's project, Here:
```
192.168.0.110:5050/cd/folder1/test1/ansible-custom-image/ansible-custom-image:v1
```
You must move the image to the destination cd/docker , because it known as docker repository in this scenario,
Dockerfile:
```
FROM ubuntu:24.04
RUN apt-get update && \
    apt-get install -y ansible rsync && \
    rm -rf /var/lib/apt/lists/*
RUN ansible-galaxy collection install ansible.posix
```
pipeline :
```
default:
  tags:
    - docker-based-pipeline-zag   # 👈 your runner’s tag
stages:
  - build
variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:v1
 # DOCKER_HOST: "unix:///var/run/docker.sock"
build_image:
  stage: build
  ##image: 192.168.0.110:5050/cd/folder1/test1/project-test1/docker:latest
  image: 192.168.0.110:5050/cd/docker/docker:latest
  before_script:
    - echo "ZZZZZZZZZZZZZZZZZZZZZZAAAAAAAAAAAAAGGGGGGGGGGG"
    - docker login -u root -p zaighoojee2eey3meiGh  192.168.0.110:5050
    #- docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
  script:
    - echo "********************************************************************"
    - docker build -t "$IMAGE_TAG" .
    - docker push "$IMAGE_TAG"
    - echo "image built"
    - echo "********************************************************************"
```

## Ansible playbooks in CICD;
Afterward that you have proper image for your ansible, now you can use it in you project,
In simple scenario I wanna use this image to create pipeline cicd whit ansible.
First you must create a public and private key in your Runner server.
and send the public key in you destination machine, than you must can ssh to the desttination server without password.
Here is the command:
```
ssh-keygen
##Copy the public key to your remote server:
ssh-copy-id root@192.168.0.119
gitlab:~ # ssh 192.168.0.119
|--------------------------------------|
|                ########              |
|            ###-----####              |
|          ###--------##-----------### |
|        ###----------#-----------###  |
|      ###-----------#----------###    |
|     ###-----------##--------###      |
|                  ####-----###        |
|                  ########            |
|______________________________________|
|             .: TOSAN :.              |
|Banking and Payment Solutions Provider|
|--------------------------------------|
Authorized uses only. All activity may be monitored and reported.
Last login: Thu Nov  6 12:28:10 2025 from 192.168.0.110
SLES15-SP3-ENT:~ #
```
Then the pipeline is like:
```
stages:

  - deploy-dev

  # - deploy-prod

  

deploy-dev:

  stage: deploy-dev

  image: 192.168.0.110:5050/cd/docker/ansible-custom-image:v1

  variables:

    ANSIBLE_HOST_KEY_CHECKING: 'false'

    ANSIBLE_FORCE_COLOR: 'true'
##inside of the container ansible running
  before_script:

    - mkdir -p ~/.ssh
##push private key to the destination machine 
    - echo "$DEPLOYER_PRIVATE_KEY" >> ~/.ssh/id_rsa

    - chmod 600  ~/.ssh/id_rsa

    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config

    - ls

    - pwd

  script:

    - cd ansible && ansible-playbook -i inventory/servers-dev.ini deploy.yml --become --become-method=sudo -t dev #-vvv
```

Artifact:
```
  artifacts:
    paths:
      - build/
    expire_in: 1 week
    when: always
```
|Key|Description|
|---|---|
|`paths:`|Defines which files or directories will be  stored as artifacts.|
|`expire_in:`|Controls how long GitLab keeps the artifact (e.g., `1 week`, `2 days`, `3 hrs`).|
|`when:`|When to upload artifacts (`on_success`, `on_failure`, or `always`).|
|`dependencies:`|Ensures the next job (e.g., `deploy_job`) can download artifacts from `build_job`.|


## Creating docker registry (NEXUS)
First install nginx:
```
sudo apt update
sudo apt install nginx -y
sudo systemctl status nginx
sudo systemctl start nginx
sudo systemctl enable nginx

```
Than install docker:
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
and than
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Now install nexus with docker compose:
docker-compose.yml
```
version: '3.4'
services:
  nexus:
    image: sonatype/nexus3:3.86.0-ubi
    hostname: nexus
    container_name: nexus
    restart: always
    ports:
      - '127.0.0.1:8081:8081'
      - '127.0.0.1:8082:8082'
    environment:
      - nexus-weapp-context-path=/nexus
    volumes:
    - /var/registry/nexus-data:/nexus-data


```
than install nexus:
```
docker compose up -d 
```
it will restart :
you must give permission on /data :
```
chown 200:200 /data/registry
chown 200:200 /data/registry/data
```
than restart the service:
```
docker compose down
docker compose up -d 
```
it takes time...

For using a nginx as reverse proxy, you must write nginx config:
```
server {
        listen 80 ;
        server_name registry.mohsen.ir;
        location / {
              proxy_pass http://localhost:8081;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto $scheme;
        }
}
```
than soft link
```
ln -s /etc/nginx/sites-available/nexus /etc/nginx/sites-enabled/nexus
unlink /etc/nginx/sites-enabled/default
```
restart nginx:
and than you will redirect nginx to the nexus on port 80 

For admin password :
```
root@registry-nexus:/etc/nginx/sites-available# docker exec -it 22d4a63ed7a8 bash
bash-5.1$ cd /nexus-data/
bash-5.1$ ls
admin.password  downloads      javaprefs  restore-from-backup
blobs           elasticsearch  keystores  tmp
db              etc            log
bash-5.1$ cat admin.password
3f5ede31-a5c9-44dc-b0a7-76b334cd237bbash-5.1$

```
change password and create a repository.
name --> http port 8082 --> enable docker V1

For pushing images :
```
server {
        listen 80 ;
        client_max_body_size 3000M;
        server_name docker.mohsen.ir;
        location / {
              proxy_pass http://localhost:8082;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto $scheme;
        }
}

```

and create a link and restart nginx
```
ln -s /etc/nginx/sites-available/docker /etc/nginx/sites-enabled/docker
systemctl restart nginx.service
```

SO FAR 8082 used for image and 8081 for UI

login:

root@registry-nexus:/etc/nginx/sites-available# docker login docker -u admin -p admin
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Error response from daemon: Get "https://docker/v2/": dial tcp 192.168.0.113:443: connect: connection refused
root@registry-nexus:/etc/nginx/sites-available#

## Docker Swarm CICD Pipeline
in this scenario we will have 3 nodes for swarm cluster.
Our pipeline contain build and deploy on the cluster.
swarm1,2,3 are three nodes for cluster and nexus as container registry.
the project that provide infra for docker swarm servers:
`http://192.168.0.110/cd/folder1/test1/ansible-playbooks/project-06`
First I use root user to deploy but for useing user mohsen:
`usemode +aG docker mohsen`
and logout and login --> So your user will have access to the docker commands.

```
docker swarm init --advertise-addr 192.168.0.114
```
Note you must delete `live-restore ` from `/etc/docker/daemon.json` :
Error response from daemon: --live-restore daemon configuration is incompatible with swarm mode.

and than gives you command for joining other servers:
```
docker swarm join --token SWMTKN-1-2wznnx1r0235tgzna10045zyc9uemd1y7966340q75lqceow4e-8i8a867zei4nlmyzkwgn0rkfo 192.168.0.114:2377
```
In master node:
```
docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
eugmgksynpkbv7isf9bqwbvtp *   swarm1     Ready     Active         Leader           28.5.2
qwhn7pa3uwwc0nay3pz8rse4y     swarm2     Ready     Active                          28.5.2
8iycy2938lvhzjri11vd5xaqj     swarm3     Ready     Active                          28.5.2
```

Docker has API. We use this API to connect via cicd pipeline.

First built and push image to registry and than to the local nexus,
	add unsecure registry to the runner server :
```
{
  "log-level": "warn",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  } ,
  "insecure-registries": [
     "192.168.0.110:5050","docker.mohsen.ir"
                    ] ,
     "live-restore": true
}
```

pipeline to push in two separate container registry.
```
workflow:
  rules:
    # Don’t run the pipeline if only README.md changed
    - changes:
        - README.md
      when: never
    # Run only for the main branch
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    # Otherwise, never run
    - when: never
default:
  tags:
    - docker-based-pipeline-zag   # 👈 your runner’s tag
stages:
  - build
  #- deploy 
build_image:
  stage: build
  image: 192.168.0.110:5050/cd/docker/docker:latest
  before_script:
    - echo "#######LET'S GO!!!!!!########"
  script:
    - echo "********************************pushing to DOCKER HUB************************************"
    - docker login -u "$DOCKER_HUB_USER" -p "$DOCKER_HUB_PASSWORD"
    - docker build -t mohsensarbaz/ubuntu-loop-script:$CI_PIPELINE_ID .
    - docker push mohsensarbaz/ubuntu-loop-script:$CI_PIPELINE_ID
    - echo "****************image pushed to Docker HUB***************"
    - echo "********************************************************************"
    - echo "****************Now pushing to local registry**************************"
    - docker login -u "$NEXUS_USER" -p "$NEXUS_PASSWORD" "$NEXUS_REGISTRY"
    - docker build -t $NEXUS_REGISTRY/ubuntu-loop-script:$CI_PIPELINE_ID .
    - docker push $NEXUS_REGISTRY/ubuntu-loop-script:$CI_PIPELINE_ID
    - echo "********************************************************************"
```

**Deploy Section**
In this section i wanna deploy the image that I have created previously,
in this regard i have to made ssh connection from my runner server to the swarm nodes, So you must use private key of runner as variable in gitlab and also you previously add public key  (`ssh-copy-id root@192.168.0.114`) to the destination servers.
```
DEPLOYER_PRIVATE_KEY
jdhskjdhksjdhskjdh...
```
so you must use this before script:
```
  before_script:

    - mkdir -p ~/.ssh
    - echo "$DEPLOYER_PRIVATE_KEY" >> ~/.ssh/id_rsa
    - chmod 600  ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - ls
    - pwd
    - echo "It's OK!!! Let's do it :)"
```

a default variables:
```
    - unset DOCKER_HOST
```
to work with swarm (connection and ....):
docker context create --> creates a context and than we use it to connect to swarm cluster :)
this is like a kubeconfig that we can connect to the cluster with it :)
```
- docker context create my-swarm --docker "host=ssh://root@192.168.0.114"
    - docker context use my-swarm
    - docker node ls ## with the context we cam connect to the swarm cluster
```
Now remove the latest from the tag of image :) it's like versioning the docker compose
```
    - sed -i "s/latest/$CI_PIPELINE_ID/g" docker-compose.yml
```
and you need need :)
```
  needs:
    - job: build_image
```
When you want to run a job manually :
```
  when:
    manual
```

## Contenting Gitlab to Kubernetes
First you must install kunernetes KUBEADM or RKE it does not matter
[[Install kubernetes 1.31 on ubuntu 24.04]]
after you installed kubernetes cluster you must connect your private registry to it.
[[Connect Cluster to the local registry]]
so in your project you have 2 steps .
first you create the proper image with your docker file and than in deploy stage you deploy your project on klubernetes cluster.
you need an image as runner that has kubectl command and also feed the gitlab a kubeconfig as a key.

## Kubernetes Concepts 
Deploymant :


Services:


configmap: 


Ingress :

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
```
helm pull ingress-nginx/ingress-nginx
```
```
tar -xzvf ingress-nginx-4.14.0.tgz
```
make this changes :
```
  hostNetwork: true
  ## Use host ports 80 and 443
  ## Disabled by default
  hostPort:
    # -- Enable 'hostPort' or not
    enabled: true
```
this opens port 80 and 443 on all hosts.
change the kind from deployment to daemonset, it's better for you :)
```
  kind: DaemonSet
```
change service--> type  to ClusterIP from (LoadBalancer)
```
    type: ClusterIP
```

```
helm upgrade --install -n ingress-nginx ingress-nginx ./ingress-nginx -f ./ingress-nginx/values.yaml --create-namespace
```

add ingress to your deployment:
```
kubectl get ingressclasses.networking.k8s.io -A
```

```
root@node2:/home# kubectl get ingressclasses.networking.k8s.io -A
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       24m
```
in ingress we point it to our service
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: devops-website
  namespace: prod
  labels:
    name: devops-website
    instance: devops-website
spec:
  ingressClassName: nginx
  rules:
    - host: "app-dev.mohsen.ir"
      http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:  # the name of service that you have
                name: devops-website
                port:
                  number: 80
```

## Backup From Gitlab 
you can take backup in two ways:
1-manuall
2-in docker-compose 



1-
```
docker exec -it gitlab /bin/bash
girlab-backup create
```
take backup in `/opt/gitlab/data/data/backups`
IMPORTANT:
arning: Your gitlab.rb and gitlab-secrets.json files contain sensitive data
and are not included in this backup. You will need these files to restore a backup.
```
gitlab:/opt/gitlab/data/config # ll
total 188
-rw------- 1 root root  16316 Nov 17 15:50 gitlab-secrets.json
-rw-r--r-- 1 root root 143506 Nov  2 08:07 gitlab.rb
-rw------- 1 root root    513 Oct 29 08:00 ssh_host_ecdsa_key
-rw-r--r-- 1 root root    179 Oct 29 08:00 ssh_host_ecdsa_key.pub
-rw------- 1 root root    411 Oct 29 08:00 ssh_host_ed25519_key
-rw-r--r-- 1 root root     99 Oct 29 08:00 ssh_host_ed25519_key.pub
-rw------- 1 root root   2602 Oct 29 08:00 ssh_host_rsa_key
-rw-r--r-- 1 root root    571 Oct 29 08:00 ssh_host_rsa_key.pub
drwxr-xr-x 2 root root   4096 Oct 29 08:01 trusted-certs
```

For restore.

1-First install fresh gitlab.
2- copy .tar file to `data/backups`
3.insode the new gitlab cd `/var/opt/gitlab/backups
4-`chown git:git 12132323-gitlab-backup.tar`
now you must stop two of services of gitlab:
5-
```
gitlab:/opt/gitlab/data/config # docker exec -it gitlab_gitlab_1 /bin/bash
root@6056359dbcae:/# gitlab-ctl status
run: gitaly: (pid 267) 16257s; run: log: (pid 264) 16257s
run: gitlab-exporter: (pid 387) 16246s; run: log: (pid 386) 16246s
run: gitlab-kas: (pid 345) 16249s; run: log: (pid 342) 16249s
run: gitlab-workhorse: (pid 347) 16249s; run: log: (pid 343) 16249s
run: logrotate: (pid 4418) 1856s; run: log: (pid 262) 16257s
run: nginx: (pid 346) 16249s; run: log: (pid 338) 16249s
run: postgresql: (pid 315) 16251s; run: log: (pid 314) 16251s
run: puma: (pid 337) 16249s; run: log: (pid 336) 16249s
run: redis: (pid 266) 16257s; run: log: (pid 265) 16257s
run: registry: (pid 379) 16247s; run: log: (pid 378) 16247s
run: sidekiq: (pid 344) 16249s; run: log: (pid 341) 16249s
run: sshd: (pid 33) 16283s; run: log: (pid 32) 16283s
```
`puma` and `sidekiq`
```
gitlab-ctl stop puma
gitlab-ctl stop sidekiq
```
6-Now you can restore
```
gitlab-backup restore BACKUP=1212121212 # without gitlab-backup
```
now you have the database, But you need ` gitlab.rb and gitlab-secrets.json `
7-
```
cp /var/gitlab-backup/gitlab.rb /opt/gitlab/data/config/
cp /var/gitlab-backup/gitlab-secrets.json  /opt/gitlab/data/config/
```
than
8-
`docker exec -it gitlab /bin/bash`
```
gitlab-ctl status 
### you have two services that are down
gitlab-ctl start 
```
Now check your gitlab with this command:
```
gitlab-rake gitlab:check SANETIZE=true
```

