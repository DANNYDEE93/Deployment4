

Step 1: Create VPC in AWS to preconfigure route table and internet gateway

	Go to VPC in AWS -->
	**diagram**

Step 2: Git code through remote repository through VS code and then commit and push changes to 	the local repository on Github.
	Uptdate Jenkinsfile with following script in order to __________________:

pipeline {
agent any
stages {
stage ('Build') {
steps {
sh '''#!/bin/bash
python3 -m venv test3
source test3/bin/activate
pip install pip --upgrade
pip install -r requirements.txt
export FLASK_APP=application
'''
}
}
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
}
}
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
}
}
stage ('Deploy') {
steps {
keepRunning {
sh '''#!/bin/bash
pip install -r requirements.txt
pip install gunicorn
python3 -m gunicorn -w 4 application:app -b 0.0.0.0 --daemon
'''
}
}
}
}
}


Step ** git code steps 


	**git commit timeline diagram*


 

Step 3: Launch instance with t2 medium capacity 
	Connect to new VPC created through Network Settings 
	Select public subnet in us-east-1a availbility zone
	Create security group with ports: 80 & 8080 [HTTP], 8000, and 22[SSH]

Step 4: Install Python 3.10 version to read python files in application code.

**sudo apt update**

Step 5:	Jenkins 2.4.01 along with "Pipeline Keep Running Step" 	plugin, 
	Important to know: Jenkins comes out with weekly releases of its installation code so previous methods for installing may not work after some time. You can visit the Jenkins User Handbook here to [learn more]!(https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)
	
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins.service 
sudo systemctl status jenkins [checks that jenkins package is actively running on EC2 before you run your build through the jenkins web browser later on]

**sudo apt update**

Installation of Java which is the language that Jenkins is written and understood from: 

sudo apt update
sudo apt install openjdk-17-jre
java -version

If you have previous installed packages for Jenkins, you will need to remove any and all files with the **rm command** and run *sudo apt clean** to clear your cached history before trying to install a new version of Jenkins and for it to work




**sudo apt update**

Step 6:	Nginx 

sudo apt-get install nginx
sudo systemctl start nginx service

edit the configuration file and **cd** into default file through the following 	path: 	"/etc/nginx/sites-enabled/default"

###First change the port from 80 to 5000, see below:
server {
listen 5000 default_server;
listen [::]:5000 default_server;

####Now scroll down to where you see “location” and replace it
with the text below:

location / {
proxy_pass http://127.0.0.1:8000;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}


**sudo apt update**


Step 7: Configure Amazon Cloudwatch monitoring agent on server 

	Need to attach permissions to AWS so that cloudwatch can be used on instance
	Go to Iam Roles: **Create** IAM roles, Trusted entity type: **AWS service**. Use case: EC2, **Next**
	Add permissions policies: Select **CloudWatch AgentServerPolicy** or AdminPolicy for GET & PUT parameters (*to allow cloudwatch agent to collect logs of info about deployment) [gives agent ability to recieve info and write to the logs to utilize them and allows metric data to be accessed by cloudwatch to get the information that the agent finds on EC2 instance **Next**
	Role name: "CloudWatchAgentServerRole" = allows EC2 instances to call AWS services on your behalf to automatically create logs of events, errors and other analytical measures of the functionality of the instance and other applications provisioned through it for the deployment process
	**Create Role**
	
Step 8:	Attach role to EC2-->
		Go to EC2 --> select the public EC2 instance created for deployment [does not tneed to be running for this step]--> Go to Actions --> Go to security --> go to **Modify IAM role** --> Choose Cloudwatchagentserverole --> Select **Update IAM Role**

Step 9:	Start EC2 instance --> Download the CloudWatch agent package --> *download link depends on platform being used to run EC2 (Current OS: Ubuntu)-->  *further instruction on downloading CloudWatch can be found [here]!(https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/download-cloudwatch-agent-commandline.html)*


**sudo apt update**


Step 10: Download the CloudWatch agent on the server that already has Python, Jenkins, and Nginx installed on it.

#download link for cloudwatch agent compatability with Ubuntu OS system
**wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb** 

#download Debian package on the server that is being run on Linux
**sudo dpkg -i -E ./amazon-cloudwatch-agent.deb**

#change to directory containing package with cloudwatch agent and other files needed to work on instance
**cd /opt/aws/amazon-cloudwatch-agent/bin/**

#use configuration manager,that comes with AWS cloudwatch as a service so that we don't have to configure the package manually
**sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard** ((why sudo and not cd)

Step 11: You should see the configuration manager open up. You need to answer the questions pertaining to the configurations of your EC2:

[[]]]

For the conditions of CloudWatch agent, I chose: 
Linux --> EC2 --> root (as user to run the agent on because ) --> 
No for 'StatsD daemon' and 'CollectD' (they are monitoring processing tools but we dont need them because we're going to use Cloudwatch) --> 
**Yes** to monitor host metrics such as CPU and memory --> 
**Yes** to monitor cpu metrics per core --> 
**Yes** to add EC2 dimensions necessary for applicaiton --> 
**Yes**to aggregate EC2 dimensions -->  
Select [4] for metrics to be collected every 60seconds --> 
Select **2** for standard configuration of our json output file where cloudwatch agent is being confiugred on --> 
Select **Yes** to accept the configuration provided (you can select **no** to edit the configurarion or change the file later) --> 
Select **no** to having any existing CloudWatch Log Agent --> 
Select **yes** to monitor log files --> Provide path to log files: '/var/logs/syslog' --> Enter to accept path default --> Enter to accept default stream name or change it --> Select the number of logs you want to be saved in a day which will allocate the necessary time for the instance to relay and collect these logs in a day. Log files can be long so choose a number that your instance can handle --> 
Add or ignore any other log files you may or may not want to collect --> 

SN..: Since I do not have permission to give AWS permission to write to my log files, I wont be able to see my config file for the cloudwatch agent in my AWS account but it is saved in my instance and I can still utilize CloudWatch. If you have permission for PUT parameter in your AWS account, you can continue:

Click **Yes** to store the JSON config file in the SSM parameter (this is possible because of the PUT parameter or write capability given in the Admin policy in the role created for the cloudwatch agent --> 
Press **Enter** to accept default of parameter store name [AmazonCloudWatch-linux], default region that the virtual server of the instance was launched in [us-east-1]--> 
**Choose or Enter** AWS credential that should be used to send the config file to the parameter store --> 
Program should exit now


Step 12: You should be at this path in the instance still: **/opt/aws/amazon-cloudwatch-agent/bin/** --> if you press **ls** you should see the config file that we just generated, sent, to the parameter store, and stored onto our EC2 instance.

	Run command **sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json** to process the amazon cloudwatch agent
	Check to see if it is running: **sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status**

  
  

Step 13:  Create staging environment in Jenkins:

	Copy and paste public ip address of EC2 instance into a new browser with port 8080 that we included in the security group when launching the instance: <http://3.95.193.197:8080>
	You will be asked to enter a password which was saved in your instance when installing Jenkins. You can find it by running the command: **cd /var/lib/jenkins/secrets/initialAdminPassword**
*don't skip over creating login credentials so that you can log into Jenkins later. 
	
	Go to **Manage Jenkins** in the Jenkins Dashboard, navigate to **Plugins** --> **Available plugins** --> Search and install **Pipeline Keep Running Step** plugin. This plugin allows me to define my script so its easier to see when the build is running properly or has been triggered. It's also easier to keep record and check historical changes in the functionality of an application. 


Step 14:  Start the multibranch pipeline on the Jenkins web browser to test the application in the GitHub repository.

	Go to **New Item** on the Dashboard --> Name project --> Select **Multibranch pipeline** --> Fill in display name --> Go to **Branch Sources** --> Select **Git** --> Under **Credentials** --> Add **Jenkins**

	You will be prompted to enter your GitHub credentials to connect GitHib repo and follow steps to make token on GitHub for Jenkins credential password **()**

Press New Item tab to create new pipeline build --> Name pipeline --> Create multibranch pipeline --> Select GitHub through branch source --> Add Jenkins source and use GitHub credentials and token. [multibranch pipeline allows the implementation of different Jenkins files for different branches of the same project which you will need for later steps in this process where I update my Jenkins file in my repo]

Switch to Github --> 9. Go back to newly created GitHub repository and copy the HTTP code or URL of the repository --> Paste code into "Repository line" in Jenkins.

Create token for Jenkins using GitHub account:

Go back to your GitHub and press profile picture/icon,

Select Settings-->Developer Settings-->Personal Access Tokens--> Tokens(classic)

Select Generate new token-->Generate new token (classic)-->Sign into GitHub if prompted

Create note description-->Select scopes: repo and admin repo to give full access to repository.

Switch back to Jenkins --> 5. Click Generate token--> Copy and paste link into password line in Jenkins

Continue on Jenkins-->Add token associated with repository--> [the token is for added security and authorization and to connect GitHub repo with Jenkins to scan and test build for application]

Step 15: Scan repository Now to test build --> Select Build History to view console output with git commands and pipeline stage process --> Pass staging environment in Jenkins before proceeding.

Check console output responses and check the phases of testing and passing the staging environment.




Step 16: Copy and paste public ip address and port 5000 (this port is necessary for nginx and we added in the nginx config file) in a new browser to run the deployment through the nginx extension that we installed on the server <ip_address:5000>

If you try to do this step, you will see that it doesn't work because we did not add port 5000 on the security group that we added to the EC2 instance. 

***get nginx browser to work**

Step 17: Go to **security group** created for the instance --> Add Port 5000 and re-try Step 16.

The web application should be available now 





Step 18: Configure CloudWatch alarm & email notifications: 

To create an alarm using the Amazon EC2 console
Open the Amazon EC2 console at https://console.aws.amazon.com/ec2/.

In the navigation pane, choose Instances.

Select the instance and choose Actions, Monitor and troubleshoot, Manage CloudWatch alarms.

On the Manage CloudWatch alarms detail page, under Add or edit alarm, select Create an alarm.

For Alarm notification, choose whether to turn the toggle on or off to configure Amazon Simple Notification Service (Amazon SNS) notifications. Enter an existing Amazon SNS topic or enter a name to create a new topic.

For Alarm action, choose whether to turn the toggle on or off to specify an action to take when the alarm is triggered. Select an action from the dropdown.

For Alarm thresholds, select the metric and criteria for the alarm. For example, you can leave the default settings for Group samples by (Average) and Type of data to sample (CPU utilization). For Alarm when, choose >= and enter 0.80. For Consecutive period, enter 1. For Period, select 5 minutes.

(Optional) For Sample metric data, choose Add to dashboard.

Choose Create.


Go to **Metrics** tab --> **CWAgent** (installed on instance) --> Select **cpu metrics** or any other metric you want to look at --> You should see the metrics displayed in a visual format at the top --> You can change the type of visual by choosing an option under *****

To create email notifications for Alarm:

After creating visuals on the cpu_usage_user metric, I wanted to be notified when the cpu usage hits a certain percentage. 

Go to Cloudwatch --> Go to 'in alarm' --> **Create alarm**
Go back to **Metrics** then **CWAgent** --> Click instance you want alarms for --> Go to **Select Metrics** --> Information should be filled out for the most part --> Modify **Statistic** and **Period** for what metric you want to be notified about and the period time that the metric is being measured --> 

	Under **COnditions**, select when conditions that the instance needs to meet in order for you to recieve a notification. I selected **Anomaly detection**, and selected **greater/equal** to 75% (bc i want to know before cpu usage hits an abnormal percentage and 75% is already high. This can help with early detection if there is an issue with functionality of my EC2 instance. --> Press **Next**

	Select **In alarm** --> Create a new topic: **Name topic** or select default for SNS connection --> Enter **email address** under email endpoints --> **Create topic** --> Press **Next** --> Fill out name and description --> Press *Next** --> Go over configurations for alarm --> **Create alarm**

auto scale or create ec2 actions so that in case certain metrics reach a specifiec threshold in the alarm, the instance can perform an action to intervene with any issues that may occur

Do a stress test in the terminal by installing: sudo apt install stress -ng. This command helps test the functionality by directing simulated traffic to the instance and seeing if it can handle it, as well as, test if my alarm is working.
 	Then run: **sudo stress-ng --matrix 1 -t 1m** to test the durability of the instance that is hosting applications needed for my deployment. This command does a 1 minute test but you can change the amount of time as needed. Since I have already configured an alarm for this instance, I recieved an email notification that my alarm was in **'in Alarm'**. This would prompt me to go to my instance and check on its processes and see if there are any configurations that need tobe done to ensure the proper functionality of my deployment. 



Recieved email and easily accessed alarm metrics since we added it to the Dashboard

Check CPU usage again. 



*After running my Jenkins build a few times and installing all the applications on my EC2 instance, it had some connectivity issues. It was performing at a slower pace but still worked well enough for my deployment. Luckily, I used a t2.medium EC2 instance becuase my usual t2.micro instance would not have been able to handle all the installations we did on it. For long term, I would need to eventually switch to an instance with a larger capacity just in case I need to install additional applications or perform more complicated processes. This issue is important to note especially for understanding business needs and the scalability of their business infrastructure. 
	
	
