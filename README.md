## <ins>Deployment 4: Using AWS VPC & EC2 instance for Jenkins Staging & Gunicorn Production </ins>
_________________________________________________
##### Danielle Davis
##### October 2, 2023
______________________________________
### <ins>PURPOSE:</ins>
___________________
&emsp;&emsp;&emsp;&emsp;	To provision an EC2 instance within my own VPC and install the necessary applications for a Jenkins staging environment with python code with Flask and Gunicorn as a proxy server for the web application. 

&emsp;&emsp;&emsp;&emsp;	Using my own VPC gave me control over IP address ranges, subnets, route tables, and gateways for consistent connectivity to the web applicaiton. It also gave me control over permissions around inbound(ingress traffic) and outbound (egress traffic) rules for added security. Similar to how a CDN balances requests coming through network traffic between the client server and web server, Nginx was used as my reverse proxy server and load balancer for forwarded HTTP requests to the web application on the Gunicorn server. 

&emsp;&emsp;&emsp;&emsp;	In provisioning the staging environment, the Jenkinsfile of the application code gave my web application access to the servers through Nginx, Gunicorn, and Flask working together to enable a daemon process or automated background system process in the Gunicorn server. With processes balancing out in the backend and frontend, this detaches the processes from each other to run independently at the same time for faster processing of the Nginx and Gunicorn servers. This also reduces the response time between requests and lowers latency when accessing my web application. 

&emsp;&emsp;&emsp;&emsp;	In previous deployments, I have used AWS Elastic Beanstalk with preconfigured settings and services to deploy my applications. In this deployment, I am using Nginx and Gunicorn servers to buffer HTTP requests and deploy my web application. Customizing my web application stack compared to Elastic Beanstalk is more a more complex process and involves more configuration control because I get to set my own permissions, standards, and metrics. Additionally, it is more beneficial because it allows greater flexibility and optimization for my web application. 

### <ins>ISSUES:</ins>
___________________
* Git Code: I tried to push my changes after I merged but realized I needed to fetch the changes in my main first to update the changes in both the remote and local repo so that they could both recognize them and accept the changes. 
* Jenkins Installation: I was having trouble installing Jenkins because I was using an old code application that did not work anymore. I needed to go to Jenkins site and find the updated code to install on my EC2. I had to remove the Jenkins packages that weren't working and clean my cache history for the working code to properly install the packages for Jenkins.
* Gunicorn deployment issues: When trying to open the web application in a new web browser, I got an error that the site could not be reached. After removing and installing Nginx on my EC2 instance multiple times, I realized I needed to rebuild my application in Jenkins to configure changes done in the Jenkinsfile and nginx config file. The rebuild basically updated the dependencies and cleared my cache history of any old builds to create a clean virtual production environment for my web application. After running a successful rebuild, I used the <ipaddress:8000> and then it worked. 


### <ins> **STEPS FOR WEB APPLICATION DEPLOYMENT** </ins>

_____________________________________________________________________________
### Step 1: Create VPC in AWS:
__________________________________________________________________________
	
* Go to VPC in AWS -->  Choose **VPC & more** to preconfigure route table and internet gateway 
	
![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/vpcDeployment4.jpg)

______________________________________________________________________________
### Step 2: Git commits
__________________________________________________________________________

* I used git code through remote repository in VS code provisioned on a separate instance and made changes to the Jenkinsfile in a second branch to double check the changes before adding, committing, and pushing those changes to my local repository on Github.
  
* Add GitHub URL in config file to give code editor permission to update my local repo. 

	
![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4remoterepo.jpg)
_______________________________________________________________________

<ins> **Additions to Jenkinsfile through VS code editor** </ins>

The changes I made to the Jenkinsfile includes the dependencies to sustain the virtual environment for the build stage, save the test stage results, and delete old builds and running processes attached to them. The script also installs dependencies for Gunicorn and Flask applications to run as a HTTP web server that runs python applications. The Gunicorn server can then run as a daemon or an automated dormant background process to handle client requests when necessary, preventing the server from getting overwhelmed.

____________________________________________________________________________

stage ('Build') {
steps {
sh '''#!/bin/bash
python3 -m venv test3
source test3/bin/activate
pip install pip --upgrade
pip install -r requirements.txt
export FLASK_APP=application

________________________________________________

stage ('test') {
steps {
sh '''#!/bin/bash
source test3/bin/activate
py.test --verbose --junit-xml test-reports/results.xml
'''
}
post{
always {
junit 'test-reports/results.xml'
}
__________________________________________________________

stage ('Clean') {
steps {
sh '''#!/bin/bash
if [[ $(ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2) != 0 ]]
then
ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2 > pid.txt
kill $(cat pid.txt)
exit 0
fi
'''
__________________________________________________________________________

stage ('Deploy') {
steps {
keepRunning {
sh '''#!/bin/bash
pip install -r requirements.txt
pip install gunicorn
python3 -m gunicorn -w 4 application:app -b 0.0.0.0 --daemon
'''
_______________________________________________________________________


* When a **git pull** is done, the server will provide you with a token of sorts to enter in GitHub to give GitHub permission to connect with the git commits and code editor. You will see these commits in your local repo as you can see below.


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4localrepo.jpg)



<ins> **Below is a git commit timeline to keep track of the changes made in my application files for deployment:** </ins>


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/gitCommitTimeline.jpg)

________________________________________________________________________

       
### Step 3: Launch instance with t2 medium capacity with the necessary protocols & applications

__________________________________________________________________________



* Press **Instances** in the Dashboard --> Press **Launch Instance** button--> Name web server --> Select **Ubuntu** for OS --> Select **t2.medium** --> Select suggested key pair -->

**Edit** Network settings --> **Select** new VPC -->	Select public subnet in us-east-1a availbility zone
**Create** security group with ports: 80 & 8080 [HTTP], 8000, 22[SSH], & 5000[Nginx] --> Press **Launch Instance**


 _______________________________________________________________________________________
 

### Step 4: Install Python 3.10 version to read python files in application code.

__________________________________________________________________________



<ins> Run commands in EC2 terminal to download newest versions and packages:</ins>

*#update system and checks for upgrades*
**sudo apt update** 

*#upgrade any applications that need upgrading*
  **sudo apt upgrade** 

*#for the python virtual environment*
**sudo apt install python3.10-venv** 

*#includes the dependencies needed for the packages being installed*
**sudo apt install python3-pip** 

______________________________________________________________________________________

### Step 5: Install Jenkins 2.4.01 

__________________________________________________________________________


* Important to know: Jenkins comes out with weekly releases of its installation code so previous methods for installing may not work after some time. You can visit the Jenkins User Handbook [here](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)

<ins> Run commands in EC2 terminal to download newest versions and packages for Jenkins 2.4.01 :</ins>

**sudo apt update**

**curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null**
**echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null**

**sudo apt-get update**

**sudo apt-get install jenkins**

**sudo systemctl start jenkins.service**

*#checks that jenkins package is actively running on EC2 before you run your build through the jenkins web browser later on*
**sudo systemctl status jenkins** 

* If you have previous installed packages for Jenkins, you will need to remove any and all files with the **rm command** and run *sudo apt clean** to clear your cached history before trying to install a new version of Jenkins and for it to work.

</ins> Installation of Java which is the language that Jenkins is written and understood from: </ins>

**sudo apt update**

**sudo apt install openjdk-17-jre**

**java -version** *(check that you have the latest version installed)*


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4jenkinsconfig.jpg)

* After taking the ip address and using port 8080 in another browser, you will be asked for a password that is installed in your instance upon installation. You can find it at the path shown below. 

* Once you create credentials for Jenkins/ staging environment --> Go to the **Manage Jenkins** tabs in the Dashboard --> **Plugins** --> **Install** the **Pipeline Keep Running Step** for continuous testing that can be triggered through git commits since it is connected with my local GitHub repo. 

___________________________________________________________________


### Step 6: Install Nginx, a reverse proxy server to act as a web server to deploy application 

_______________________________________________________________________

**sudo apt update**

**sudo apt-get install nginx**

**sudo systemctl start nginx service**

</ins> **Edit** the configuration file and **cd** & **nano** into default file through the following path: "/etc/nginx/sites-enabled/default" </ins>

* Use **sudo chmod 777 default** to give permissions to write into file

**First change the port from 80 to 5000 so that HTTP requests can be heard on port 5000 through the Nginx reverse proxy server and Flask can be utilized for Gunicorn to deploy application:**
server {
listen 5000 default_server;
listen [::]:5000 default_server;

**Now scroll down to where you see “location” and replace with the text below so that requests are forwarded to port 8000 for the Gunicorn proxy server  :**
location / {
proxy_pass http://127.0.0.1:8000;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/nginxdep4.jpg)

______________________________________________________________________


### Step 7: Configure Amazon CloudWatch monitoring agent on server 

__________________________________________________________________________

</ins> Before installing, you need to configure permissions for AWS so that CloudWatch can be used on the instance: </ins>
	
**Go to IAM Roles** --> **Create** IAM roles, Trusted entity type: **AWS service** --> Use case: EC2, **Next**
	
* Add permissions policies: Select **CloudWatch AgentServerPolicy** or AdminPolicy for GET & PUT parameters [gives agent ability to receive info and write to info logs to assess them and allows metric data to be accessed by CloudWatch to send info to the agent on EC2 instance] --> **Next**
  
* Role name: "CloudWatchAgentServerRole" [allows EC2 instances to call AWS services on your behalf to automatically create logs of events, errors, and other analytical measures of the functionality of the instance and other applications provisioned through it for the deployment process] --> **Create Role**

	
* Attach role to EC2 on AWS dashboard: Select the public EC2 instance created for deployment --> Go to Actions --> Go to security --> Go to **Modify IAM role** --> Choose Cloudwatchagentserverole --> Select **Update IAM Role**

* Go to EC2 terminal --> Download the CloudWatch agent package --> *Download link depends on platform being used to run EC2 (Current OS: Ubuntu)-->  *further instruction on downloading CloudWatch can be found [here](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/download-cloudwatch-agent-commandline.html)*

____________________________________________________________________________

<ins> **Run commands to download the CloudWatch agent on the EC2 instance:** </ins>

**sudo apt update**

*#download link for cloudwatch agent compatability with Ubuntu OS system*
**wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb** 

*#download Debian package on the server that is being run on Linux*
**sudo dpkg -i -E ./amazon-cloudwatch-agent.deb**

*#change to directory containing package with cloudwatch agent and other files needed to work on instance*
**cd /opt/aws/amazon-cloudwatch-agent/bin/**

*#use configuration manager,that comes with AWS cloudwatch as a service so that we don't have to configure the package manually*
**sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard** 

*You should see the configuration manager open up*

__________________________________________________________________________________________


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4cloudwatchagent.jpg)

 
</ins> **Answer the questions pertaining to the configurations of your EC2:** </ins>

*For the conditions of CloudWatch agent, I chose:* 

* Linux --> EC2 --> root user --> 
**Select** No for 'StatsD daemon' and 'CollectD' [monitoring processing tools] --> 

* **Yes** to monitor host metrics such as CPU and memory --> 

* **Yes** to monitor cpu metrics per core --> 

* **Yes**to aggregate EC2 dimensions -->  

* Select for metrics to be collected every 20 -60 seconds --> 

* Select **2** for standard configuration of our json output file where the CloudWatch agent is being configured on --> 

* Select **Yes** to accept the configuration provided (you can select **no** to edit the configuration or change the file later) --> 

* Select **No** to having any existing CloudWatch Log Agent --> 

* Select **Yes** to monitor log files --> Provide path to log files: **/var/logs/syslog** --> Enter to accept path default --> Enter to accept default stream name or change it --> Select the number of logs you want to be saved in a day which will allocate the necessary time for the instance to relay and collect these logs in a day. [Log files can be long so choose a number that your instance can handle] --> Add or ignore any other log files you may or may not want to collect --> 

&emsp;&emsp;&emsp;&emsp;	*Note:* Since I do not have permission to give AWS permission to write to my log files, I wasn't able to see my config file for the CloudWatch agent in my AWS account but it is saved in my instance and I can still utilize CloudWatch. If you have permission for a PUT parameter in your AWS 	account, you can continue:

* Click **Yes** to store the JSON config file in the SSM parameter (this is possible because of the PUT parameter or write capability given in the Admin policy in the role created for the CloudWatch agent --> 

* Press **Enter** to accept default of parameter store name [AmazonCloudWatch-linux], default region that the virtual server of the instance was launched in [us-east-1]--> 

**Choose or Enter** AWS credential that should be used to send the config file to the parameter store --> Program should exit now

* *You should be at this path in the instance still:*  **/opt/aws/amazon-cloudwatch-agent/bin/** --> if you press **ls** you should see the config file that we just generated, sent to the parameter store, and stored onto our EC2 instance.
  
* Run command: **sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json** to process the amazon cloudwatch agent

* Check to see if agent is running properly: **sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status**
  
__________________________________________________________________________

### Step 8:  Create staging environment in Jenkins:

__________________________________________________________________________

* Copy and paste public ip address of EC2 instance into a new browser with port 8080 that we included in the security group when launching the instance: <http://3.95.193.197:8080>
  
* You will be asked to enter a password which was saved in your instance when installing Jenkins. You can find it by running the command: **cd /var/lib/jenkins/secrets/initialAdminPassword**
*don't skip over creating login credentials so that you can log into Jenkins later. 
	
* Go to **Manage Jenkins** in the Jenkins Dashboard, navigate to **Plugins** --> **Available plugins** --> Search and install **Pipeline Keep Running Step** plugin. This plugin allows me to define my script so it's easier to see when the build is running properly or has been triggered. It is also easier to keep records and check historical changes in the functionality of an application because it automates the build process. 

______________________________________________________________________________

</ins> Start the multibranch pipeline on the Jenkins web browser to test the application in the GitHub repository: </ins>

* Go to **New Item** on the Dashboard --> Name project --> Select **Multibranch pipeline** --> Go to **Branch Sources** --> Select **GitHub** under credentials --> Add **Jenkins** [multibranch pipeline allows the implementation of different Jenkins files for different branches of the same project which you will need for later steps in this process where I update my Jenkins file in my repo]

*You will be prompted to enter your GitHub credentials to connect GitHub repo and follow steps to make token on GitHub for Jenkins credential password.* 

*Switch back to GitHub. Copy and paste repo URL into "Repository line" in Jenkins.*

</ins> **Create token for Jenkins using GitHub account:** </ins>

* Go back to your GitHub and press profile picture/icon --> Select Settings--> Developer Settings--> Personal Access Tokens--> Tokens(classic) --> 	Select scopes: repo and admin repo to give full access to repository --> Generate new token (classic)--> Sign into GitHub if prompted --> Copy 		and paste token into password line in Jenkins [the token is for added security and authorization and to connect GitHub repo with Jenkins to scan 	and test build for application]

* Scan repository Now to test build --> Select **Build History** to view console output with git commands and pipeline stage process --> Pass staging environment in Jenkins before proceeding --> Check **console output** responses and check the phases of the staging environment.

![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4jenkinsbuild.jpg)


__________________________________________________________________________

### Step 9: Nginx, Gunicorn and Flask Production Environment 

__________________________________________________________________________

* Copy and paste public ip address and port 8000 (this port is necessary for Nginx and we added in the Nginx config file) in a new browser to run the deployment through the nginx extension that we installed on the server <ip_address:8000>


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/urlshortener.jpg)


* Nginx acts as a load balancer and reverse proxy server/middleman between the EC2 instance and the web application. It creates a level of security by buffering requests and only allowing necessary responses from the backend. This also handles traffic from overloading the web application server.
* Gunicorn, installed with the code in our application code, acts as a proxy server running on port 8000 adding into the configuration file in Nginx. Gunicorn was installed in the application code changes I made in the Jenkinsfile. The flask application uses python with Gunicorn to create a framework or translation of the python function calls into HTTP responses so that Gunicorn can access the endpoint which was the application URL.

   
__________________________________________________________________________

### Step 10: Configure CloudWatch alarm & email notifications: 

__________________________________________________________________________


</ins> **To create a CloudWatch alarm using the Amazon EC2 console:** </ins>
  

* Go to CloudWatch --> Go to 'in alarm' --> **Create alarm** --> Click instance you want alarms for --> Information should be filled out for the most part --> Modify **Statistic** and **Period** for what metric you want to be notified about and the period time that the metric is being measured --> Press **Select Metrics**

* Under **Conditions**, select when conditions that the instance needs to meet for you to receive a notification --> For Alarm thresholds, select the metric and criteria for the alarm. You can leave the default settings for selections or customize Group samples and Type of data. For Alarm when, choose threshold %, for period: the amount of time you want the metrics to be measured --> Press **Next**

* Select **In alarm** --> Create a new topic: **Name topic** or select default for SNS connection --> Enter **email address** under email endpoints --> **Create topic** --> Press **Next** --> Fill out name and description --> Press *Next** --> Go over configurations for alarm --> **Create alarm**

* Select the instance and **choose** Actions, **Monitor and troubleshoot**, **Manage CloudWatch alarms** --> Select alarm to attach alarm to EC2 instance running applications for deployment.


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4cloudwatchconfig.jpg)


</ins> **Test CloudWatch alarm:** </ins>
 
* Do a stress test in the terminal by installing: **sudo apt install stress -ng** [This command helps test the functionality by directing simulated 	traffic to the instance and seeing if it can handle it as well, as test if my alarm is working.]
 	
* Then run: **sudo stress-ng --matrix 1 -t 1m** [to test the durability of the instance that is hosting applications needed for my deployment] This command does a 1 minute test but you can change the amount of time as needed. Since I have already configured an alarm for this instance, I received an email notification that it was in a state of **'in Alarm'**. This would prompt me to go to my instance and check on its processes and see if there are any configurations that need to be done to ensure the proper functionality of my deployment. 

* Received email and easily accessed alarm metrics since we added it to the Dashboard


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4cloudwatchnotify.jpg)


______________________________________________________________________________

### <ins>RE-BUILD & DEPLOY:</ins>
__________________________________________________________________



![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/rebuild%26alarm.jpg)

&emsp;&emsp;&emsp;&emsp;		As I explained above in my **Issues** section and **Step 9**, I had to rebuild my staging environment in Jenkins for my application to deploy. Re-running the build in Jenkins ensured that all changes and dependencies were refreshed and updated,  as well as that the cache history was cleared out old builds. Most importantly, it ensures a greater level of optimization because by testing in Jenkins, I can identify any errors in the code before pushing the build to the production environment.

&emsp;&emsp;&emsp;&emsp;		For my CloudWatch alarm, I chose to measure the **CPU User Usage** and set the alarm to notify me when the usage gets over 75% and adjusted the percentage to assess the notification process. I saw that the CPU usage went all the way up to 80% after rebuilding the application in Jenkins due to all  the processing power that its using to complete so many different tasks at once.

&emsp;&emsp;&emsp;&emsp;			After running my Jenkins build a few times and installing all the applications on my EC2 instance, it had some connectivity issues. It was performing at a slower pace but still worked well enough for my deployment. Luckily, I used a t2.medium EC2 instance because my usual t2.micro instance would not have been able to manage all the installations I added to it. For the long term, I would need to eventually switch to an instance with a larger capacity just in case I need to install additional applications or perform more complicated processes. This issue is important to note especially for understanding business needs and infrastructure scalability. 


_____________________________________________

### <ins>SYSTEM DIAGRAM:</ins>
_________________________________________________


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/newdep4.jpg)

_________________________________________________

### <ins>OPTIMIZATION:</ins>
_____________________________________________
Jenkins was particularly important in the optimization and error handling for the deployment. Below are some ways I could have had better optimization with my deployment:

*	Using a webhook to integrate the changes and automatically trigger the rebuild in Jenkins to make sure the proxy server is up and running for the web application to work properly.
*	
<ins> **Lowering the processing power:** </ins>

*	Configuring another EC2 to split up the processing power that my t2 medium EC2 instance was having trouble with handling. My EC2 is running slow now and having connectivity issues for my application and is in danger of shutting down if I have to make more configuration to my coding or have to add more applications to the instance. 
*	Automating the action of turning on a backup EC2 instance or auto scaling in the CloudWatch alarms setting when memory and CPU utilization went above 25-50%. This gives me security around the functioning of my applications installed on the server because I have a backup plan before my EC2 is in any more danger of shutting down. 
