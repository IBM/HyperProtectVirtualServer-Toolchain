# Deploy software to private Hyper Protect Virtual Servers using IBM toolchain
Build, test, and deploy on custom servers with the IBM Delivery Pipeline
## Introduction
With IBM Cloud Continuous Delivery, you can build, test, deploy, and manage apps with toolchains. IBM Cloud Continuous Delivery includes delivery pipelines, making these tasks automated, which in turn, requires little to no human interference.

In this tutorial, you’ll learn how to deploy on custom servers using the IBM Delivery Pipeline, which is commonly used to deploy builds on Kubernetes or Cloud Foundry. We’ll be using the demo application [AcmeAir](https://github.com/acmeair/acmeair-nodejs), a Node.js implementation of the Acme Air Sample Application. The code will be automatically integrated, dockerized on a custom-built server, uploaded to a private container registry, and deployed on multiple custom servers (in this case the Cloud IBM Hyper Protect Virtual Servers). The best part is that all of this can be done using resources available through an IBM Cloud account. Let’s get started!


## Prerequisites 
To complete this tutorial, you need the following:

* A computer to customise the Toolchain running [Docker](https://docs.docker.com/install/)

* An [IBM Cloud account](https://cloud.ibm.com/login)

* A configured [IBM Cloud Container Registry](https://cloud.ibm.com/kubernetes/registry/main/start)
   
## Estimated time
Completing this tutorial should take about 1 hour.

## Step 1: Set up your Hyper Protect Virtual Server
To begin, you need to create the server required to deploy your software on the private server. Don’t worry, setting up a Hyper Protect Virtual Server is simple. 

Begin by opening the command line interface (CLI) on your computer and generate an SSH key pair by entering the command `ssh-keygen` and following the instructions. Make sure to remember where you store your SSH keys (you may need to install [OpenSSH](https://www.ssh.com/ssh/openssh/#sec-OpenSSH-Download) before doing this). 

Now, use the generated private key to create your Hyper Protect Virtual Server [here](https://cloud.ibm.com/catalog). Try to log in to your server via the terminal to make sure everything is working properly. You can find this process explained in more detail in the [Hyper Protect Virtual Server overview](https://cloud.ibm.com/docs/services/hp-virtual-servers?topic=hp-virtual-servers-overview) as well as in the [SSH documentation](https://www.ssh.com/ssh/keygen/#sec-Creating-an-SSH-Key-Pair-for-User-Authentication).

## Step 2: Enable an SSH key login for your IBM Delivery Pipeline
To enable the toolchain to deploy on custom servers, you use a custom docker image running Linux, which stores the SSH keys. This will be used to send commands on to your server. I’ll discuss more on this later.

To generate the docker image, download the sshimage folder from [this](https://github.ibm.com/Hans-Hueppelshaeuser/HyperProtectVirtualServer_Toolchain/tree/master/docs) git repository to your desktop. If you’re using a Mac, use **Shift+Command+G** in Finder to open the Go to Folder search mask. If you’re on a Windows device, you’ll use **Windows+R**.

Enter `~/.ssh/` or enter the path shown on your terminal when you generate your SSH keys. A folder should open with three text files inside named *id_rsa*, *id_rsa.pub* and *known_hosts*. Copy all three files in this folder.

Now, open the CLI and navigate to the sshimage folder. Run the command `docker build .`. Docker creates an image with a small Linux version and your keys in place. With the command `docker image ls` you should be able to see your newly created image.

Next you need to tag `docker tag <local_image> us.icr.io/<my_namespace>/<my_repo>:<my_tag>` and upload `docker push us.icr.io/<my_namespace>/<my_repo>:<my_tag>` to your image to the IBM Container Registry. Check out the [Kubernetes Registry Quick Start guide](https://cloud.ibm.com/login?redirect=%2Fkubernetes%2Fregistry%2Fmain%2Fstart) for more details.


## Step 3: Create the pipeline
Head over to the [DevOps toolchains page](https://cloud.ibm.com/devops/getting-started?env_id=ibm:yp:eu-de) and create a Cloud Foundry toolchain. Link the git repository with the code you want to use and enter an API-Key (if you don't have one you can generate one).

After creating your toolchain, you should see a dashboard where you can configure multiple elements. Click the field with the label **Delivery Pipeline**. The two stages of your pipeline should now be visible.

Click on the gear in the upper right corner of the **Build Stage** and then click the **Configure Stage** option. Choose the custom Docker image as your builder type and enter the name of your sshimage from your IBM Cloud Container Registry.

Next, click on **Add Job** and select **Deploy**. Choose the custom Docker image as your builder type. Enter the name of your sshimage from your Cloud Container Registry. 

Now open the **Environment properties** tab. Add a secure property with the name **DOCKER_USERNAME** and the value **iamapikey**. Add a second secure property with the name **DOCKER_PASSWORD** and an API key as your value. You can generate an API key [here](https://cloud.ibm.com/iam/apikeys). Finally, add a text property with the name **SERVER1** and the IP address of your Hyper Protect Server as its value.


## Step 4: Configure the pipeline
(Please note that the following steps are specific to the Node.js version of AcmeAir but altering these steps for different programs can be done.) 

Go back to the **Jobs** tab, open your **Build Stage**, and add this to the **Build Script window**:

`ssh -t -t $SERVER_1 'rm -r acmeair-nodejs'`

`ssh -t -t $SERVER_1 'git clone https://github.com/acmeair/acmeair-nodejs.git'`

`ssh -t -t $SERVER_1 'cd acmeair-nodejs && docker build -t acmeair/web .'`

Find the **Jobs** tab once more and open your **Deploy Stage**. Now enter the following in the **Build Script window**:

`ssh -t -t $SERVER_1 'docker login -u iamapikey -p $DOCKER_PASSWORD uk.icr.io'`

`ssh -t -t $SERVER_1 'docker tag acmeair/web uk.icr.io/hansdocker/acmeair'`

`ssh -t -t $SERVER_1 'docker push uk.icr.io/hansdocker/acmeair'`

Return to your toolchain overview and press the play button on your Build Stage to see if your toolchain is working. After it’s finished, you should see the AcmeAir Docker Image in the form of a docker image in your IBM Container Registry.

Navigate back to the overview of your pipeline and click on the gear in the upper right corner of the **Deploy Stage**. Then, click on **Configure Stage**.
Open the **Environment properties** tab and add a secure property with the name **DOCKER_USERNAME** and the value **iamapikey**. 
Add a secure property with the name **DOCKER_PASSWORD** and an API key as your value. Then add a text property with the name **SERVER1** and the IP address of your Hyper Protect Server as its value.
Return to the **Jobs** tab and remove the **Rolling Deploy** stage. You now need to create four new Deploy Stages and name them *Prepare, Deploy, Availability Test*, and *Info*. Set all four to use the custom Docker image **sshimage**, just like you did in part 1. Now you’re ready to set the code for these four stages.
For the **Prepare** stage enter:

`ssh -t -t root@$SERVER 'docker stop acmeair_web_001'`

`ssh -t -t root@$SERVER 'docker rm acmeair_web_001'`

`ssh -t -t root@$SERVER 'docker ps'`

For the **Deploy** stage enter:

`ssh -t -t root@$SERVER 'docker login -u iamapikey -p $DOCKER_PASSWORD uk.icr.io'`

`ssh -t -t root@$SERVER 'docker pull uk.icr.io/hansdocker/acmeair'`

`ssh -t -t root@$SERVER 'docker run -d -P -p 9080:9080 --name acmeair_web_001 --link mongo_001:mongo uk.icr.io/hansdocker/acmeair'`

For the **Availability Test** stage enter:

`#!/bin/bash`

`if curl -ivs http://$SERVER:9080`

`then echo "Server is UP"`

`else`

`echo "Server is down" 1>&2`

`fi`

For the **Info** stage enter:

`ssh -t -t root@$SERVER 'docker ps'`

`echo http://$SERVER:9080`

Now return to your toolchain overview and click **play** on your Deploy Stage to see if your toolchain works.
After your finished, you should be able to click on **Info** in the list of your stages to view your logs. There will be a link to the AcmeAir website running on your Hyper Protect Virtual Server. You can easily add more servers by simply duplicating the Deploy Stage and changing the IP address in the environment properties to your new server.


## Summary
Congratulations! By following this tutorial, you can now use IBM toolchain to connect to various servers by using the SSH connection standard. If you’d like, check out the [IBM Toolchain deploys to Hyper Protect Virtual Servers](https://www.youtube.com/watch?v=CiP9Lvk5480&feature=youtu.be) video that demonstrates the toolchain running and explains the function of the code in more detail.  
You can also learn more about the IBM Delivery Pipeline and toolchain by visiting the [IBM Cloud documentation](https://cloud.ibm.com/docs/services/ContinuousDelivery?topic=ContinuousDelivery-deliverypipeline_about).

