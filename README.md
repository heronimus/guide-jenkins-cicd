# Guide: Jenkins CI/CD Pipeline with Github Integration on Local Environment

This guide intended for learning purposes only, originally created for Blibli Future Program (Infra team). Some parts of this guide may be not suitable for your needs, please adjust accordingly.

In this tutorial, we will set up Jenkins using Jenkins community docker images, and use Ngrok as tunnel-forwarding for your Jenkins instance. Here's what we will do through the guide:

### Table of Contents
- [Jenkins Container Setup](#jenkins-container-setup)
- [Jenkins Web-UI Setup](#jenkins-web-ui-setup)
- [Add New Worker Node to Jenkins](#add-new-node-docker-vm-to-jenkins)
- Jenkins-Github Integration:
    - [Create New Jobs](#jenkins-github-integration-create-new-job)
    - [Ngrok Tunnel Forwarding](#jenkins-github-integration-ngrok-tunnel-local-jenkins)
    - [Github Repository Configuration](#jenkins-github-integration-repo-configuration)
- [Test CI/CD Pipeline](#test-your-cicd)

----------------------

## Jenkins Container Setup

Pull Jenkins docker image:

    docker pull jenkins/jenkins:lts

Run Jenkins container (with port 8080 & 50000, run as daemon, container name: `future-jenkins`)

    docker run -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home -d --name future-jenkins jenkins/jenkins:lts

Open Firewall on your machine, port 8080 for jenkins web-ui, and port 50000 for node connector.

    firewall-cmd --add-port=8080/tcp --permanent
    firewall-cmd --add-port=50000/tcp --permanent
    firewall-cmd --reload

The Jenkins will open at port 8080! try to open that!

** Additional info about Jenkins docker: https://github.com/jenkinsci/docker/blob/master/README.md

------
## Jenkins Web-UI Setup

- Jenkins will need you to enter initial password, initial password is located at`/var/jenkins_home/secrets/initialAdminPassword` inside the container, to show the password use `docker exec`.

      docker exec future-jenkins cat /var/jenkins_home/secrets/initialAdminPassword

  Initial password will show, copy and enter password to Jenkins Web-UI.

- Jenkins will try to install sugested plugin, wait until all plugin installed.
- Enter user credential for admin.
- Done, your Jenkins is up & running now! Try to go around in the Jenkins.


## Add New Node (Docker-VM) to Jenkins
*** This part only applicable if you have another VM/Instance for running Docker, it's fine to skip this.

We will execute anything about the docker deployment on Docker-VM (seperate VM) not on Jenkins-VM/Container, so we will link Docker-VM to the Jenkins-VM.

#### Run command below on your **Docker-VM** node, not on Jenkins-VM.
- Before start, Jenkins will need your nodes agent to have Java, to install that open your **Docker-VM** then install Java.

      yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel

- Then, create jenkins-workspace directory on **Docker-VM**, then change permission for your user.

      sudo mkdir /opt/jenkins-workspace
      sudo chown -R your-user:your-user /opt/jenkins-workspace

#### Back to Jenkins

- Back to Jenkins homepage, navigate to **``Manage Jenkins`` --> `Manage Nodes and Cloud` --> `New Node`**
- On the Node name write 'Docker-VM', select as `Permanent Agent`, OK.
- Fill the form:

  ![](/docs/images/Clipboard_2020-07-26-05-52-35.png)

- Select launch method using SSH (Or you can use other option such as Jenkins Agent). Change 'Host Key Verification Strategy' to `Non verifying Verifycation Strategy`.

- Add your SSH credentials (username & password) via Credentials 'Add' button. Make sure that you used user that can be used on SSH and have permission to run Docker and open jenkins-workspace. You may need to change additional configuration (Advanced) if you have custom SSH port settings / different Java path on your target machine.

  ![](/docs/images/Clipboard_2020-07-26-06-11-36.png)

- The final config will be like this:

  ![](/docs/images/Clipboard_2020-07-26-06-14-21.png)

- Save

- Now you'll see your Docker-VM shows up as Jenkins node agent:

  ![](/docs/images/Clipboard_2020-07-26-06-30-52.png)


## Jenkins-Github Integration: Create New Job

- Prepare repository on your github account (use existing/create new).
- Add `Jenkinsfile` to your repository, contain below pipelines code for testing:
  ```
   pipeline {

    agent {
      // Run command on Docker-VM, instead inside Jenkins container.
      label 'docker-vm'
    }

    stages {
      stage ('Test Pipeline') {
        steps {
          echo "Show Docker"
          sh '''
            docker --version
            docker image ls

            echo "You can do docker-compose, and other command here!!"
          '''
        }
      }
    }

  }

    ```
  Jenkinsfile contain pipeline, which Jenkins will lookup and execute on build.
- Create `New Item` on Jenkins, give it a name, and select `Pipeline`, then OK.

  ![](/docs/images/Clipboard_2020-07-26-04-43-21.png)

- You'll see configuration option, scroll down to `Build Trigger`, then check on `GitHub hook trigger for GITScm polling`.
- Scroll down again to `Pipeline` then do this:
  - Change `Definition` to `Pipeline script from SCM`
  - Fill repository URL with your repository URL
  - FIll script path: `Jenkinsfile`
  - Save/Apply

  ![](/docs/images/Clipboard_2020-07-26-04-48-48.png)

## Jenkins-Github Integration: Ngrok Tunnel (Local Jenkins)

Github need to communicate with Jenkins services to run webhook and callback, so if Jenkins deployed on local/behind the NAT environment, tunnel fowarding will be needed. Fortunately there's some free tunnel services like Ngrok.

- Register Ngrok: https://dashboard.ngrok.com/signup
- Download Ngrok to your server:

      wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
      unzip ngrok-stable-linux-amd64.zip

- Connect Ngrok to your account, on Ngrok web dashboard copy the auth token, then run:

      ./ngrok authtoken <ngrok-token>

- Run Ngrok

      ./ngrok http 8080

    It'll show:

    ![](/docs/images/Clipboard_2020-07-26-05-21-21.png)

    See the fowarding option? now you can open Jenkins on public internet with url https://xxxxxxx.ngrok.io, try open on your browser!

## Jenkins-Github Integration: Repo Configuration

After your jenkins now is publicly available, set you Github repository to connect with your Jenkins via Ngrok URL.
- Open your Github repository settings, open `Webhooks`, then click `Add Webhook`.

  ![](/docs/images/Clipboard_2020-07-26-05-27-17.png)

- Fill the form:
  - Payload URL = your Ngrok url (use the https one!) + /github-webhook/.
                   For example: https://5649a3b388a7.ngrok.io/github-webhook/
  - Content type = application/json
  - Secrets = Get from Jenkins homepage go to People -> Admin -> Configure -> API Token -> Add New Token, then copy token to 'Secrets' field on github.

  ![](/docs/images/Clipboard_2020-07-26-07-18-49.png)

  - Save

## Test your CI/CD

- Try to push anything to your git repository, see if you pipelines works!
  ![](/docs/images/Clipboard_2020-07-26-07-31-31.png)

- Edit the pipeline, suit the pipeline for your CI/CD jobs like: build docker images, run docker-compose, scale up/down your nodes.

- Pipeline example for deploy app using docker-compose: [Jenkinsfile](/Jenkinsfile)
