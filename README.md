## <ins>Deployment 4: Using AWS for VPC & EC2 instance for Jenkins Staging & Nginx Production </ins>
_________________________________________________
##### Danielle Davis
##### October 2, 2023
______________________________________
### <ins>PURPOSE:</ins>
___________________
&emsp;&emsp;&emsp;&emsp;

### <ins>ISSUES:</ins>
___________________
&emsp;&emsp;&emsp;&emsp;		



### <ins> **STEPS FOR WEB APPLICATION DEPLOYMENT** </ins>

_____________________________________________________________________________
Step 1: Create VPC in AWS:
	Go to VPC in AWS -->  Choose **VPC & more** to preconfigure route table and internet gateway 
	
![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/vpcDeployment4.jpg)

______________________________________________________________________________
Step 2: Use git code through remote repository through VS code by creating a second branch to make changes in the Jenkinsfile from before then adding, committing, and pushing those changes to my local repository on Github.

	
![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4remoterepo.jpg)
_______________________________________________________________________

	Below are important additions that I changed in the Jenkinsfile to include the dependencies to sustain the virtual environment for the build stage, saves the test stage results, deletes old builds and running processes attached to them. The script also installs dependecies for  Gunicorn with the Flask application to run as a HTTP web server that runs python applications. The web server can then run as a daemon or an automated dormant background process to handle client requests when necessary preventing the server from getting overwhelmed.

_______________________________________________________________________________________-

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



![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4localrepo.jpg)

________________________________________________________________________

       
Step 3: Launch instance with t2 medium capacity with the necessary protocols & applications

 Press **Instances** in the Dashboard --> Press **Launch Instance** button--> Name web server --> Select **Ubuntu** for OS --> Select **t2.medium** --> Select suggested key pair -->

  **Edit** Network settings --> **Select** new VPC -->	Select public subnet in us-east-1a availbility zone
	**Create** security group with ports: 80 & 8080 [HTTP], 8000, 22[SSH], & 5000[Nginx] --> Press **Launch Instance**


 _______________________________________________________________________________________

Step 4: Install Python 3.10 version to read python files in application code.

<ins> Run commands in EC2 terminal to download newest versions and packages for Python 3.10:</ins>

   **sudo apt update** --> *[update system and checks for upgrades]*

   **sudo apt upgrade** -->*[upgrade any applications that need upgrading]

   **sudo apt install python3.10-venv** *[for the python virtual environment]* -->

   **sudo apt install python3-pip** *[includes the dependencies needed for the packages being installed]* -->

______________________________________________________________________________________

Step 5:	Install Jenkins 2.4.01 along with "Pipeline Keep Running Step" 	plugin, 
	*Important to know: Jenkins comes out with weekly releases of its installation code so previous methods for installing may not work after some time. You can visit the Jenkins User Handbook here to [learn more]!(https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)

<ins> Run commands in EC2 terminal to download newest versions and packages for Python 3.10:</ins>

**sudo apt update**

**curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null**
**echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null**

**sudo apt-get update**

**sudo apt-get install jenkins**

**sudo systemctl start jenkins.service**

**sudo systemctl status jenkins** *[checks that jenkins package is actively running on EC2 before you run your build through the jenkins web browser later on]*

	*If you have previous installed packages for Jenkins, you will need to remove any and all files with the **rm command** and run *sudo apt clean** to clear your cached history before trying to install a new version of Jenkins and for it to work.

</ins> Installation of Java which is the language that Jenkins is written and understood from: </ins>

**sudo apt update**

**sudo apt install openjdk-17-jre**

**java -version** *(check that you have the latest version installed)*


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4jenkinsconfig.jpg)


___________________________________________________________________


##### Step 6: Install Nginx, a reverse proxy server to act as a web server to deploy application 

**sudo apt update**

**sudo apt-get install nginx**

**sudo systemctl start nginx service**

</ins> **Edit** the configuration file and **cd** & **nano** into default file through the following path: "/etc/nginx/sites-enabled/default" </ins>

**First change the port from 80 to 5000, see below:**
server {
listen 5000 default_server;
listen [::]:5000 default_server;

**Now scroll down to where you see “location” and replace with the text below:**
location / {
proxy_pass http://127.0.0.1:8000;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/nginxdep4.jpg)

______________________________________________________________________


#### Step 7: Configure Amazon Cloudwatch monitoring agent on server 

</ins> [Before installing, need to attach permissions to AWS so that cloudwatch can be used on instance:] </ins>
	
 **Go to IAM Roles** --> **Create** IAM roles, Trusted entity type: **AWS service** --> Use case: EC2, **Next**
	
 	Add permissions policies: Select **CloudWatch AgentServerPolicy** or AdminPolicy for GET & PUT parameters [gives agent ability to recieve info and write to info logs to assess them and allows metric data to be accessed by cloudwatch to send info to the agent on EC2 instance], **Next**
  
	Role name: "CloudWatchAgentServerRole" [allows EC2 instances to call AWS services on your behalf to automatically create logs of events, errors and other analytical measures of the functionality of the instance and other applications provisioned through it for the deployment process, **Create Role**
	
	Attach role to EC2 on AWS dashboard: Select the public EC2 instance created for deployment --> Go to Actions --> Go to security --> Go to **Modify IAM role** --> Choose Cloudwatchagentserverole --> Select **Update IAM Role**

	Go to EC2 terminal --> Download the CloudWatch agent package --> *Download link depends on platform being used to run EC2 (Current OS: Ubuntu)-->  *further instruction on downloading CloudWatch can be found [here]!(https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/download-cloudwatch-agent-commandline.html)*

____________________________________________________________________________

<ins> [Run commands to download the CloudWatch agent on the EC2 instance:] </ins>

**sudo apt update**

#download link for cloudwatch agent compatability with Ubuntu OS system
**wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb** 

#download Debian package on the server that is being run on Linux
**sudo dpkg -i -E ./amazon-cloudwatch-agent.deb**

#change to directory containing package with cloudwatch agent and other files needed to work on instance
**cd /opt/aws/amazon-cloudwatch-agent/bin/**

#use configuration manager,that comes with AWS cloudwatch as a service so that we don't have to configure the package manually
**sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard** 

*You should see the configuration manager open up*

__________________________________________________________________________________________


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4cloudwatchagent.jpg)

 
</ins> Answer the questions pertaining to the configurations of your EC2: </ins>

*For the conditions of CloudWatch agent, I chose:* 

Linux --> EC2 --> root user --> 
**Select** No for 'StatsD daemon' and 'CollectD' [monitoring processing tools] --> 

**Yes** to monitor host metrics such as CPU and memory --> 

**Yes** to monitor cpu metrics per core --> 

**Yes**to aggregate EC2 dimensions -->  

Select for metrics to be collected every 20 -60 seconds --> 

Select **2** for standard configuration of our json output file where cloudwatch agent is being confiugred on --> 

Select **Yes** to accept the configuration provided (you can select **no** to edit the configurarion or change the file later) --> 

Select **No** to having any existing CloudWatch Log Agent --> 

Select **Yes** to monitor log files --> Provide path to log files: **/var/logs/syslog** --> Enter to accept path default --> Enter to accept default stream name or change it --> Select the number of logs you want to be saved in a day which will allocate the necessary time for the instance to relay and collect these logs in a day. [Log files can be long so choose a number that your instance can handle] --> Add or ignore any other log files you may or may not want to collect --> 

* Note: Since I do not have permission to give AWS permission to write to my log files, I wasn't able to see my config file for the cloudwatch 		agent in my AWS account but it is saved in my instance and I can still utilize CloudWatch. If you have permission for PUT parameter in your AWS 	account, you can continue:

Click **Yes** to store the JSON config file in the SSM parameter (this is possible because of the PUT parameter or write capability given 		in the Admin policy in the role created for the cloudwatch agent --> 

Press **Enter** to accept default of parameter store name [AmazonCloudWatch-linux], default region that the virtual server of the 			instance was launched in [us-east-1]--> 

**Choose or Enter** AWS credential that should be used to send the config file to the parameter store --> Program should exit now


* You should be at this path in the instance still:  **/opt/aws/amazon-cloudwatch-agent/bin/** --> if you press **ls** you should see the config file that we just generated, sent, to the parameter store, and stored onto our EC2 instance.

	Run command: **sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json** to process the amazon cloudwatch agent
	Check to see if agent is running properly: **sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status**
  

##### Step 8:  Create staging environment in Jenkins:

	a.Copy and paste public ip address of EC2 instance into a new browser with port 8080 that we included in the security group when launching the instance: <http://3.95.193.197:8080>
  
	b. You will be asked to enter a password which was saved in your instance when installing Jenkins. You can find it by running the command: **cd /var/lib/jenkins/secrets/initialAdminPassword**
*don't skip over creating login credentials so that you can log into Jenkins later. 
	
	c. Go to **Manage Jenkins** in the Jenkins Dashboard, navigate to **Plugins** --> **Available plugins** --> Search and install **Pipeline Keep Running Step** plugin. This plugin allows me to define my script so its easier to see when the build is running properly or has been triggered. It's also easier to keep record and check historical changes in the functionality of an application. 

______________________________________________________________________________

</ins> Start the multibranch pipeline on the Jenkins web browser to test the application in the GitHub repository: </ins>

	Go to **New Item** on the Dashboard --> Name project --> Select **Multibranch pipeline** --> Go to **Branch Sources** --> Select **GitHub** under credentials --> Add **Jenkins** [multibranch pipeline allows the implementation of different Jenkins files for different branches of the same project which you will need for later steps in this process where I update my Jenkins file in my repo]

You will be prompted to enter your GitHub credentials to connect GitHib repo and follow steps to make token on GitHub for Jenkins credential password. 

*Switch back to GitHub. Copy and paste repo URL into "Repository line" in Jenkins.*

</ins> Create token for Jenkins using GitHub account: </ins>

	Go back to your GitHub and press profile picture/icon --> Select Settings--> Developer Settings--> Personal Access Tokens--> Tokens(classic) --> 	Select scopes: repo and admin repo to give full access to repository --> Generate new token (classic)--> Sign into GitHub if prompted --> Copy 		and paste token into password line in Jenkins [the token is for added security and authorization and to connect GitHub repo with Jenkins to scan 	and test build for application]

Scan repository Now to test build --> Select **Build History** to view console output with git commands and pipeline stage process --> Pass staging environment in Jenkins before proceeding --> Check**console output** responses and check the phases of the staging environment.

![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4jenkinsbuild.jpg)


##### Step 9: Copy and paste public ip address and port 5000 (this port is necessary for nginx and we added in the nginx config file) in a new browser to run the deployment through the nginx extension that we installed on the server <ip_address:5000>

If you try to do this step, you will see that it doesn't work because we did not add port 5000 on the security group that we added to the EC2 instance. 

Go to **security group** created for the instance --> Add Port 5000 and re-try Step 16.

The web application should be available now 

[]!()




#### Step 10: Configure CloudWatch alarm & email notifications: 

</ins> To create an alarm using the Amazon EC2 console: </ins>
  

Go to Cloudwatch --> Go to 'in alarm' --> **Create alarm** --> Click instance you want alarms for --> Information should be filled out for the most part --> Modify **Statistic** and **Period** for what metric you want to be notified about and the period time that the metric is being measured --> Press **Select Metrics**

Under **Conditions**, select when conditions that the instance needs to meet in order for you to recieve a notification 
 
 	For Alarm thresholds, select the metric and criteria for the alarm. You can leave the default settings for selections or customize   Group samples by (Average) and Type of data to sample (CPU utilization). For Alarm when, choose >= and enter 0.80. For Consecutive period, enter 1. For Period, select 5 minutes --> Press **Next**

Select **In alarm** --> Create a new topic: **Name topic** or select default for SNS connection --> Enter **email address** under email endpoints --> **Create topic** --> Press **Next** --> Fill out name and description --> Press *Next** --> Go over configurations for alarm --> **Create alarm**


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4cloudwatchconfig.jpg)


*For the future, you can auto scale or create ec2 actions so that in case certain metrics reach a specifiec threshold in the alarm, the instance can perform an action to intervene with any issues that may occur*

Select the instance and choose Actions, Monitor and troubleshoot, Manage CloudWatch alarms. --> Select alarm 

 
Do a stress test in the terminal by installing: **sudo apt install stress -ng** [This command helps test the functionality by directing simulated 	trafic to the instance and seeing if it can handle it, as well as, test if my alarm is working.]
 	
  Then run: **sudo stress-ng --matrix 1 -t 1m** [to test the durability of the instance that is hosting applications needed for my deployment] This command does a 1 minute test but you can change the amount of time as needed. Since I have already configured an alarm for this instance, I recieved an email notification that it was in a state of **'in Alarm'**. This would prompt me to go to my instance and check on its processes and see if there are any configurations that need tobe done to ensure the proper functionality of my deployment. 


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/dep4cloudwatchnotify.jpg)


Recieved email and easily accessed alarm metrics since we added it to the Dashboard


### <ins>SYSTEM DIAGRAM:</ins>


![](https://github.com/DANNYDEE93/Deployment4/blob/main/static/deployment4Diagram.jpg)



### <ins>OPTIMIZATION:</ins>
___________________
&emsp;&emsp;&emsp;&emsp;	After running my Jenkins build a few times and installing all the applications on my EC2 instance, it had some connectivity issues. It was performing at a slower pace but still worked well enough for my deployment. Luckily, I used a t2.medium EC2 instance becuase my usual t2.micro instance would not have been able to handle all the installations I added to it. For long term, I would need to eventually switch to an instance with a larger capacity just in case I need to install additional applications or perform more complicated processes. This issue is important to note especially for understanding business needs and the scalability of their business infrastructure. 
	
	
