# AWS EC2 Start Stop Automation
## Overview
The objective of this framework is to provide an Automation process to start and start EC2 instances and instance applications on an automated schedule. Primarily to reduce infrastructure costs on out of hours compute.<br/>
The framework provides 3 SSM Runbooks:
* **Start-AutoGroup** – Starts VM’s and applications in groups
* **Stop-AutoGroup** – Stops VM’s and applications in groups
* **Run-AutoCommand** – Called by the start and stop Runbooks to call a command on the VM to facilitate the start or stop of applications on that host

EC2 Instances are tagged as follows:
* AutoGroup <StringList> - Group this instance belongs to
* AutoCommand <String> - An optional host command that is called with either parameter ‘start’ or ‘stop’ during execution of Runbook

This framework was designed to automated the start and stop of SAP hosts and instances but can be adapted to none SAP systems

## Tag EC2 Instances
In this example the following test EC2 instances have been created:
![image](https://github.com/user-attachments/assets/54fa9671-a441-42cf-87a7-36ec4291b2d7)

The instances are given the following tags:
|Server|AutoGroup|AutoCommand|
|------|---------|-----------|
|MYDBSERVER|DBGroup|/usr/sap/awscontrol.sh|
|MYAPPSERVER1|APPGroup||
|MYAPPSERVER2|APPGroup|C:\usr\sap\awscontrol.ps1|

*In this example MYAPPSERVER1 doesn't have any apps that need starting as part of the process so the AutoCommand tag is not specified*

For Example:

![image](https://github.com/user-attachments/assets/cbf17110-3ed7-4c2d-b0e8-0ab15874b528)

## VM Prioritisation
VM's are grouped by tag AutoGroup we provide the start and stop group order when calling the Start/Stop-AutoGroup runbook.

**Example 1**<br/>
A three tier BW SAP system can be defined as follows:

|VM|Usage|AutoGroup|
|--|-----|---------|
|BWDBHOST|BW Database Host|DBGroup|
|BWASCSHOST|BW ASCS Host|DBGroup|
|BWAPPHOST1|BW Instance|APPGroup1|
|BWAPPHOST2|BW Instance|APPGroup1|
|BWAPPHOST3|BW Instance|AppGroup1|

We can use Start-AutoGroup with groups: DBGroup,APPGroup1 to start the VM's in order. The DB and ASCS hosts are started first simultaneously followed by the 3 app instances

We use Stop-AutoGroup with groups: APPGroup1, DBGroup to stop the VM's in order. The 3 app instances are stopped first simultaneously followed by the DB and ASCS instances


**Example 2**<br/>
Expanding on Example 1, a Business Objects and Business Objects Data Services systems are added to the landscape, these system are dependent on the BW system
We have also added an AutoCommand to be called as part of the start or stop process

|VM|Usage|AutoGroup|AutoCommand|
|--|-----|---------|-----------|
|BWDBHOST|BW Database Host|DBGroup|/usr/sap/azcontrol.sh|
|BODBHOST|BO Database Host|DBGroup|C:\usr\sap\azcontrol.ps1|
|DSDBHOST|DS Database Host|DBGroup|C:\usr\sap\azcontrol.ps1|
|BWASCSHOST|BW ASCS Host|DBGroup|C:\usr\sap\azcontrol.ps1|
|BWDAPPHOST1|BW Instance Host|APPGroup1|C:\usr\sap\azcontrol.ps1|
|BWDAPPHOST2|BW Instance Host|APPGroup1|C:\usr\sap\azcontrol.ps1|
|BWDAPPHOST2|BW Instance Host|APPGroup1|C:\usr\sap\azcontrol.ps1|
|BOAPPHOST|BO SIA+WEB|APPGroup2||
|DSAPPHOST|DS SIA+WEB|APPGroup2||

*No AutoCommand is set for BOAPPHOST and DSAPPHOST because the BO services are configured to auto start*

Start-AutoGroup: DBGroup,APPGroup1,APPGroup2

Stop-AutoGroup: APPGroup2, APPGroup1, DBGroup

## Create Automation Runbooks
Use AWS Systems Manager to create the 3 Automation Runbooks
![image](https://github.com/user-attachments/assets/a677dd5a-0c81-438f-be95-d31826a26430)

## Test Runbook
In this example run the Stop-AutoGroup Runbook to test the stopping of the EC2 instances
![image](https://github.com/user-attachments/assets/6e2391d6-30e0-44f2-85fe-bb3a7ea50c06)
We want to stop the APPGroup servers first and then the DBGroup server.

We are not including app processing because the application control script are not setup on each server yet

During execution if we look at EC2 instances we can see that the two app servers are stopping first
![image](https://github.com/user-attachments/assets/1527984d-631e-4ba9-91bc-44a92187a934)

After the two app servers are stopped the database instance then stops:
![image](https://github.com/user-attachments/assets/0e3e10b9-e975-4145-b5c3-2e21a640081e)

Run the Start-AutoGroup Runbook to start up the EC2 instances

## Setup Application Control Scripts
The start and stop Runbooks can call a local host command specified by tag:AutoCommand with parameter either ‘start’ or ‘stop’ to control the start and stop of applications and services on that VM.

The command is called either as a PowerShell script or as a shell script depending on platform type

Example host application control scripts can be found at https://github.com/dave-carter-uk/Apps-Stop-Start-Control

For this example we are just using simple scripts that don’t do anything:

**MYDBSERVER (Linux)**<br/>
![image](https://github.com/user-attachments/assets/faf2228c-6918-45da-b285-0498372ca931)

**MYAPPSERVER1/2 (Windows)**<br/>
![image](https://github.com/user-attachments/assets/8492bb14-f28c-42f0-ae07-455b9670ad79)

As the EC2 instances are currently running we can run the Start-AutoGroup Runbook with the option to just run the app startup
![image](https://github.com/user-attachments/assets/e03e869b-46db-4afe-8489-70af39c91abb)

After the Runbook completes drill down into the execution steps to see the output from the RunShellScript and RunPowerShellScript steps:
![image](https://github.com/user-attachments/assets/9df3cfa2-f9e1-4dd3-b9b7-036e1178286a)

So far we are running Runbooks to start and stop EC2 instances and applications by groups manually, next step is to get the process to run on a schedule.

There are a number of ways of scheduling a Runbook to run at a specific time in the Examples below SSM Maintenance Window is used

## Create Maintenance Role
First create a role to be used by the Maintenance Window<br/>
IAM -> Roles -> Create Role<br/>
**Trusted Identity Type**: AWS Service<br/>
**Use Case**: Systems Manager<br/>
Add Permissions:<br/>
* AmazonEC2FullAccess<br/>
*	AmazonSSMAutomationRole<br/>
* AmazonSSMMaintenanceWindowRole<br/>

Provide Role a name and create. We will use this role later in our Maintenance Window

## Create Mainenance Window
SSM -> Maintenance Window -> Create Maintenance Window

**Name:** StartNoneProdSystems (example)<br/>
**CRON/Rate expression:** cron(0 8 ? * 2-6 *)<br/>
*The CRON definition specifies to run this Maintenance Window at 8am on Days Monday - Friday*.<br/>

Add a Task to the new Maintenance Window of type Register Automation Task<br/>
Provide a task name<br/>
**Add document:** Start-AutoGroup<br/>
**Use Default version**<br/>
**Targets:** not required<br/>
Provide Runbook parameters:<br/>
![image](https://github.com/user-attachments/assets/dcd2b58b-97f7-4af6-8a29-882655b48ba5)

This task will run the Start-AutoGroup runbook at 8am (Mon-Fri) and start both VM Instances and Apps on our select Instances.<br/>
Instances tagged with AutoGroup:DBGroup will be processed first, followed by Instances tagged with AutoGroup:APPGroup.

Provide our new role
![image](https://github.com/user-attachments/assets/16dcb5a7-40ce-4805-b066-a40a322d60f1)

Save and wait for the Maintenance Window to run

## Evaluate Maintenance Window
After the Maintenance Window has completed we can drill into the task details to see the steps ran and their status:
![image](https://github.com/user-attachments/assets/1e1c1565-7669-4c50-a7e8-30e70442680c)

Step name: ProcessVMandApp was ran, steps: ProcessVMOnly and ProcessAppOnly where not ran – because we defined this task to include both VM’s and Apps

Two StartVMGroup and StartAppGroup steps where ran because we defined this task for 2 AutoGroup’s

Drill down further into each processed step to view the output, for example from the AutoCommand ran on the DBGroup server:
![image](https://github.com/user-attachments/assets/eed72d2f-371d-4211-bd08-6a982c4640ec)

We can now set up a similar Maintenance Window and Task to stop our applications and instances at a given schedule.

Using a Stop and Start schedule we can elect to run our Instances only between a specified number of hours on required days.












