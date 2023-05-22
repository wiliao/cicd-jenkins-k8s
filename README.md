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

