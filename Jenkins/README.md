# Working with Jenkins

[Jenkins](https://jenkins.io/) is a very popular Java-based open source continuous integration (CI) server that allows teams to continuously build applications across platforms.

Visual Studio Team Services (VSTS) includes Team Build, a native CI build server that allows compilation of applications on Windows, Linux and Mac platforms. However, it also integrates well with Jenkins for teams who already use or prefer to use Jenkins for CI.

There are two ways to integrate VSTS with Jenkins

* One way is to completely replace VSTS Build with Jenkins. This involves the configuration of a CI pipeline in Jenkins and a web hook in VSTS to invoke the CI process when source code is pushed by any member to a repository or a branch. The VSTS Release Management will be configured to connect to the Jenkins server through the configured Service Endpoint to fetch the compiled artifacts for the deployment.

* The alternate way is to use Jenkins and Team Build together. In this approach, a Jenkins build will be nested within the VSTS build. A build definition will be configured in the VSTS with a **Jenkins** task to queue a job in Jenkins and download the artifacts produced by the job and publish it to the VSTS or any shared folder from where it can be picked by the Release Management. This approach has multiple benefits -

    1. End-to-end traceability from work item to source code to build and release
    1. Triggering of a Continuous Deployment (CD) when the build is completed successfully
    1. Execution of the build as part of the branching strategy

This lab covers both the approaches and the following tasks will be performed

* Provision Jenkins on Azure VM using an Azure Marketplace Jenkins Template
* Configure Jenkins to work with Maven and VSTS. 
* Create a build definition in Jenkins
* Configure VSTS to integrate with Jenkins
* Setup Release Management in VSTS to deploy artifacts from Jenkins

## Pre-requisites

1. Microsoft Azure Account: You will need a valid and active azure account for the lab.

1. You need a **VSTS** account and a [Personal Access Token (PAT)](https://docs.microsoft.com/en-us/vsts/accounts/use-personal-access-tokens-to-authenticate)

1. You will need [Putty](http://www.putty.org/), a free SSH and Telnet client

1. You will need the **Docker Integration** extension installed and enabled for your VSTS account. You can perform this step later while using the VSTS Demo Generator.

## Setting up the VSTS project

1. Use [VSTS Demo Generator](https://vstsdemogenerator.azurewebsites.net/?name=MyShuttleDocker&templateid=77373) to provision a team project on the VSTS account
    
    ![VSTS Demo Gen](images/vstsdemogen-1.png)

1. Select **MyShuttleDocker** for the template

    ![VSTS Demo Gen](images/vstsdemogen-2.png)

    **Note:** Using the VSTS Demo Generator link will automatically select the MuShuttleDocker template in the demo generator for the team project creation. To use other project templates, use the alternate link provided below:  https://vstsdemogenerator.azurewebsites.net/

## Setting up the Jenkins VM

1. To configure Jenkins, the Jenkins VM image available on the Azure MarketPlace will be used. This will install the latest stable Jenkins version on an Ubuntu Linux VM along with the tools and plugins configured to work with Azure.

    <a href="https://portal.azure.com/#create/azure-oss.jenkinsjenkins" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
    </a>

1. Once the machine is provisioned, click **Connect** and note down the \<username> and \<ip address>. We will need this to connect to the VM from ***Putty***

    ![SSH Connection Info](images/vmconnect_ssh1.png)

1. Jenkins, by default, listens on port 8080 using HTTP. If you want to set up HTTPS communication, you will need to provide an SSL certificate. If you do not setup HTTPS communication, the best way to make sure the sign-in credentials are not leaked due to a "Man-in-the-middle" attack is to only log in using SSH tunneling. An SSH tunnel is an encrypted tunnel created through an SSH protocol connection, which can be used to transfer unencrypted traffic over an unsecured network. Simply run this command from a command prompt.
    ````cmd
    putty.exe -ssh -L 8080:localhost:8080 <username>@<ip address>
    ````
    ![Connecting from Putty](images/ssh2.png)

    **Note:** You should have Putty.exe in the path or provide an absolute path of the putty.exe 


1. Login with the user name and password that you provided when you provisioned the VM.

1. Once you are connected successfully, open a browser on your host machine and type [http://localhost:8080](http://localhost:8080). You should see the Jenkins page. 

1. For security reasons, Jenkins will generate a password and save it in a file on the server. You will need to provide that initial password to unlock Jenkins.

   ![Jenkins Initial Password](images/jenkinsinitialemptypwd.png)

   **Note:** **At the time of writing this lab, an open issue in Jenkins was noted where the setup wizard would not resume after restart, skipping some of the steps listed below. If you do not see the screen above, steps 5 to 7 will not work. The workaround is to use the default user name *admin* with the initial admin password (explained in step #7 below)..**

1. Return to the **Putty** terminal and type the following command to open the log file that contains the password. Copy the password
    >sudo vi /var/lib/jenkins/secrets/initialAdminPassword

    *You can double click on the line and use **CTRL+C** copy the text and place it in the clipboard. Press **ESC and then :q!** to exit the vi editor without saving the file*

1. Back in the browser, paste the text and select **Continue**

    ![Unlock Jenkins - First Time](images/jenkinsinitialpwd.png)

    Jenkins has a large ecosystem with a strong and active open source community users contributing several hundreds of useful plugins. When you setup Jenkins, you can start with installing the most commonly used plugins or select and install the ones that you want.

1. You will need the Maven plug-in which is not installed by default but we will do it later. For now, we will go with the suggested plugins. Select **Install suggested plugins**

    ![Customize Jenkins Plugins](images/customizejenkins-plugins.png)

1. You will need to create a new *Admin* user for Jenkins. Provide a user name and password and select **Continue**

    ![Create Admin User for Jenkins](images/firstadminuser.png)

1. Now, you have Jenkins ready to use. Select **Start using Jenkins**

    ![Jenkins Ready](images/jenkinsready.png)

## Installing and configuring Maven

 Starting from Jenkins version 2, Maven plugin is not installed by default.  You will need to do this manually

1. Select **Manage Jenkins** on the main page of the Jenkins portal.  This will take you to the Manage Jenkins page, the central one-stop-shop for all your Jenkins configuration. From this screen, you can configure your Jenkins server, install and upgrade plugins, keep track of system load, manage distributed build servers, and more!

1. Select **Manage Plugins**

    ![Manage Jenkins](images/manage-jenkins1.png)

1. Select **Available** tab and search **maven-plugin** in the filter box

1. Check **Maven Integration Plugin** and select **Install without restart** to install the plugin. Wait for the plug-in to be installed.

    ![Install Maven](images/installmavenplugin.png)

1. Select **Manage Jenkins** and select **Global Tool Configuration**

    ![Global Tool Configuration](images/manage-tools-config.png)

1. You have added the Maven plugin for Jenkins but you have not installed Maven on the machine. Jenkins provides great out-of-the-box support for Maven.  You could  manually install Maven by extracting the ***tar*** file located in a shared folder. Alternatively, you can let Jenkins do all the hard work and download Maven for you. Select the **Install automatically** checkbox. Jenkins will download and install Maven from the Apache website the first time a build job needs it.

    We will install version 3.5, the latest version at the time the lab is written

    ![Maven Installer](images/maveninstallerconfig.png)

1. Click **Apply** and select **Back to Dashboard** to return to the home page.

## Creating a new Build Job

1. From Jenkins home page, select **New Item**. Enter a name for the build definition, and select **Maven project**. Click **OK** to save

   ![](images/newbuilddef.png)

1. Scroll down to the **Source code Management** section. Select **Git** and Enter the clone URL of your VSTS Git repo. It should be in **http://\<your account name>.visualstudio.com/
 \<your project name>/_git/MyShuttle**

   ![Configuring VSTS Git URL](images/jenkins-vstsrepo.png)

   **Note:** VSTS Git repos are private. So you will need to provide the credentials to access the repository. If you have not set the Git credentials, you can do it from VSTS. Select the **Clone** link. Provide a user name and password and select **OK**

   ![Generating Git Credentials](images/vsts-generategitcreds.png)

1. Select **Add | Jenkins** to add a new credential. Enter the user name and password created in the previous step and click **Add** to close the wizard

    ![Adding Credentials to Jenkins](images/jenkinscredentials.png)

1. Select the credential from the drop-down. The error message should disappear now

   ![VSTS Git config in Jenkins](images/jenksaddvstsgit.png)

1. The source code for this application has both unit tests and UI tests. We will include the unit tests but skip the UI tests from running now.

1. Scroll down to the **Build** section. Enter the following text in the **Goals and Options** field

   >package -Dtest=FaresTest,SimpleTest

1. Click **Save** to navigate to the main page of the project you just created

   ![Build Settings in Jenkins](images/jenkins-buildsettings.png)

1. The last configuration that we will do for this lab is to add a *Post-build* action to publish the artifacts. Scroll down to **Post-Build Actions**  section, click **Add post-build action** and select **Archive the artifacts**

   ![Post Build Action](images/jenkinspostbuildaction.png)

1. Enter  **target/*.war** in the text box. Click **Save** to save the settings and return to the project page

   ![Archive War](images/jenkinsarchiveartifacts.png)

1. We have completed all the necessary configuration and Select **Build Now** to start an Ad-hoc build

1. You will notice the build progress just below the left side navigation menu

   ![Running Ad-hoc Build](images/adhocbuild.png)

1. You can select the build number to get into the details of the build including the build artifacts, in this case, the WAR file for the project.

   ![Build Details](images/builddetails.png)

   ![Build Artifacts](images/buildmodules.png)

1. Select the **Test Results** links if you want to see the results of the unit tests that were included in the build definition.

## Approach 1: Running Jenkins without VSTS Build

In this section, we will cover the first approach. We will run Jenkins separately. We will configure a service hook in VSTS to trigger a Jenkins build whenever a code is pushed to a particular branch.

1. Go to your VSTS project page and navigate to the **Admin** | **Service Hooks** page

    ![Navigate to service hooks page](images/servicehooks.png)

1. Select **Create subscription** button

1. In the *New subscriptions dialog* select **Jenkins** and click **Next**

   ![Create a new subscription](images/vsts-createjenkinsservice.png)

1. Select **Code pushed**  for the **Trigger on this of type event**

1. Choose the repository and branch and Select **Next**

   ![VSTS - Trigger Code Pushed](images/vsts-jenkinssubscription1.png)

1. In the next page, select **Trigger generic build** for the perform action field

1. Enter the URL of the Jenkins server in **http://{ip address or the host name}**  format

1. Enter the **User name** and **Password** that you have setup for Jenkins

1. Select **Test** to check the settings. If the settings are correct, click **Finish** to save and exit

   ![VSTS - Jenkins Info](images/vsts-jenkinssubscription2.png)

Now you can try making a change and commit your code. Upon commit,VSTS will notify Jenkins to initiate a new build.

## Approach 2: Wrapping Jenkins Job within VSTS Build

We will cover the second approach in this section. We will wrap the Jenkins job within a Team Build. The key benefit taking this approach is the end to end traceability from work item to code to build and release can be maintained

First, we will need to create an endpoint to the Jenkins server

1. From the **Admin | Services** tab, select the **New Service Endpoint | Jenkins** button to create a new endpoint

1. Provide the server URL and the user name and password to connect to the server. The server URL is in http://[server IP address or DNS name] format. Click **Verify Connection** to validate the entries and to confirm that VSTS is able to reach the Jenkins server

   ![Jenkins Endpoint](images/jenkinsendpoint.png)

1. Select **Builds** from the **Build and Release** help and select **+New** to create a new build Definition**

1. You will notice that there is a **Jenkins** template out-of-the-box. Select the template and click **Apply**

    ![Jenkins Template](images/jenkinsbuildtemplate.png)

1. In the *Build process* settings, select the Jenkins service endpoint that you created and enter **MyShuttle** for Jenkins Job Name.

    ![Jenkins Settings in Team Build](images/vsts-buildjenkinssettings.png)

1.  Next, select the **Get Sources** task. Since, we  are only triggering a Jenkins job from the build, there is no need to download the code to the VSTS build agent. Select **Advanced Options** and check the "Don't sync sources** option

    ![Get Sources Settings in Team Build](images/vsts-getsourcessettings.png)

1. Next, select the **Queue Jenkins Job** task. This task queues the job on the Jenkins server. Make sure that the services endpoint and the job name are correct.  You will see the two options checked- leave them as-is. 

     ![Jenkins Settings in Team Build](images/vsts-buildjenkinssettings1.png)

>The **Capture console output and wait for completion** option when selected, will capture the output of the Jenkins build console when the build runs, and also make the build  wait for the Jenkins Job to complete and return whether the job succeeded or failed. The **Capture pipeline output and wait for pipeline completion** option is very similar but applies to Jenkins pipelines (a build that has more than one job nested together). 

1. When the build is complete, the **Jenkins Download Artifacts** task will download the build artifacts produced by the Jenkins job to the staging directory

    ![Download Jenkins Artifact](images/downloadjenkinsartifact.png)

1. Finally, we will publish the artifacts to VSTS. 

1. Click **Save & queue** to save the build definition and start a new build. 

## Deploying Jenkins Artifacts with Release Management

Next, we will configure Visual Studio VSTS Release Management to fetch and deploy the artifacts produced by the build.

1. As we are deploying to Azure, we will need to create an endpoint to Azure. You should also create an endpoint to Jenkins, if you have not done it before

1. After you have created the endpoints, select the **Build & Release** hub and select **+ Create a new Release definition** to start creating a new release definition

1. We will use the **Azure App Service Deployment** template as we are trying to publish a web application

   ![New Release Definition](images/newreleasedefintion.png)

1. We will name the default environment as **Dev**

   ![New Environment](images/rm_environment.png)

1. We will link this release definition to the MyShuttle build on Jenkins. Select **+Add** to add an artifact

1. Select **Jenkins** for the *Source type*. Select the Jenkins endpoint you created above and enter **MyShuttle** for the *Source(Job)* - Note this should map to the project name that you have configured in Jenkins

1. If you have configured Jenkins server and the source correctly, you will get a message showing the output of the build, in this case it should be ***myshuttledev.war***

   ![Add Jenkins artifact](images/rm_addjenkinsartifact.png)

1. You are now ready to deploy!

1. You can refer to the [Deploying Tomcat+MySQL application to Azure with VSTS](../tomcat/) if you want to continue with the deployment.


## Logging into Jenkins with the default credentials

1. To log in to Jenkins when you did not get a chance to configure the initial admin, you can use the default user name **admin**

1. Copy and paste the password text from `\var\lib\jenkins\secrets\initialAdminPassword` 

1. To change password, click on the user name on the top-right 

1. Select **Configure**. Scroll down to the **password** section. Specify a new password and click **Save**


# Appendix

## Installing Git Plugin

1. From the main page of the Jenkins portal, select **Manage Jenkins** and then select **Manage Plugins**

    ![Manage Jenkins](images/manage-jenkins2.png)

1. Select the **Available** tab. 

1. Enter **Git plugin** in the filter textbox 

1. Select **Git plugin** in the search list and select **Install without Restart**


## Installing VSTS Private agent

1. From VSTS, select **Admin**|**Agent Queue**

1. Select **Download agent**

    ![Agent Queue](images/vsts-agentqueue.png)

1. If you are accessing this page from the VM, it should default to **Linux**. Otherwise, select the tab.

1. Click **Download** to start downloading the agent. This is typically saved in the *Downloads* folder

    ![Download VSTS agent](images/downloadvstsagent.png)

1. Open a terminal window and enter the following commands one-by-one

    ````cmd
    mkdir vstsagent
    cd vstsagent
    tar -zxvf ../Downloads/vsts-agent-linux-x64-2.126.0.tar.gz
    ````
1. Once the files are extracted, run `./config.sh` to configure the agent. You will need to enter the VSTS URL and provide your PAT

1. After you have configured, start the agent by running the following command `./run.sh`