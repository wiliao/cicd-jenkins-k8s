# cicd-jenkins-k8s
Setup CI/CD with Jenkins on Docker Desktop

## 1. Install Git on WSL2

    git --version

    git version 2.25.1

    ifconfig

    172.22.216.211

## 2. Install GitLab on WSL2

    docker search gitlab
    docker pull gitlab/gitlab-ce

    sudo mkdir -p /data/git
    sudo nano /data/git/docker-compose.yml

    cd /data/git
    docker-compose up -d

## 3. Access Gitlab from Windows

    http://172.22.216.211:8929

Check the intial password for user "root":

    docker exec -it gitlab cat /etc/gitlab/initial_root_password

Login and change the password for the first time.

![screen-shot-login-gitlab](screen-shot/login-to-gitlab.png)

## 4. Install Jenkins

    docker pull jenkins/jenkins:latest-jdk17

    sudo mkdir /data/jenkins/
    sudo nano /data/jenkins/docker-compose.yml

## 5. Configure Jenkins

    cd /var/run
    sudo chown root:root docker.sock
    sudo chmod o+rw docker.sock

    cd /data/jenkins/
    sudo mkdir /var/jenkins_home
    docker-compose up -d
    sudo chmod 777 data/
    sudo docker logs -f jenkins

 Jenkins initial setup is required. An admin user has been created and a password generated at /var/jenkins_home/secrets/initialAdminPassword

![screen-shot-login-jenkins-01](screen-shot/jenkins-initial-page.png)

![screen-shot-login-jenkins-02](screen-shot/jenkins-after-init-password.png)

![screen-shot-login-jenkins-03](screen-shot/jenkins-getting-started.png)

![screen-shot-login-jenkins-04](screen-shot/change-password.png)