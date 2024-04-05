# HOL 02: Deploying A Hybrid Infrastructure For Researchers In AWS
### HandsOn Labs on Cloud Computing and Big Data for BioMed
### Samuel Leal Rodríguez
### High-Performance And Distributed Computing For Big Data 2023 - 2024


## Introduction

Jupyter Notebooks have become an essential tool for analyzing data and disseminating findings in data science. This hands-on lab guides you through setting up a private Jupyter Notebook server on AWS for your research team. Furthermore, we will deploy a public Nginx web server using Voila to share your team's findings with the public.

## Objectives

This hands-on lab aims to introduce you to the basics of cloud computing by deploying a hybrid infrastructure for researchers in AWS. The infrastructure will include a private Jupyter Notebook server accessible via VPN and a public Voila server accessible from the Internet.

## Infrastructure Setup

The following image schematically represents the structure of the Virtual Private Cloud configuration as determined by the instructor.

![]([/hol02/images/image002.png](https://github.com/samuleal/hol02/blob/main/images/image002.png))

_Figure 1. Target VPC configuration according to instructor guidelines._

The first step that I took in the task was the creation of the VPC. In Figure 2 a quick depiction of the VPC structure is shown. As you can see, there are three different subnets, each of them connected to a different routing table. There is only one internet gateway granting access only to the Production and the DMZ subnets. The HOL02-Research subnet does not have access to the internet. Nonetheless, it is important for it to have a routing table, as it is needed to grant it access to the S3 bucket as we will discuss later.

![](RackMultipart20240405-1-b4fxau_html_3b20470db4ef1cc0.png)

_Figure 2. Overview of the created VPC. There are 3 subnets connected to 3 different routing tables. Only one internet gateway was created and one endpoint gateway to the S3 bucket was used to connect with the Research subnet._

This configuration is achieved by first creating the subnet associations (i.e. associate each routing table to a specific subnet) and then adding the routes (e.g. the internet gateway). The following image shows an example of the DMZ Subnet configuration.

![](RackMultipart20240405-1-b4fxau_html_47a350bf75d242cd.png)

_Figure 3. DMZ Subnet Routing configuration. The route to the internet gateway was manually added._

# Security groups

I created the security groups at the same time when I created the EC2 instances. This caused me some trouble (see [Troubleshooting](#Troubleshooting)), mostly for the subnet which was disconnected from the internet (HOL02-Research).

In the next table a summary of the security groups, the ports and the sources accepted as inbound traffic are displayed.

| **Instance**| **Summary of Security Group configuration**|
| --- | --- |
| **HOL02-VPN**| ![](RackMultipart20240405-1-b4fxau_html_babeeb5545906e2b.png) |
| **HOL02-VoilaServer**| ![](RackMultipart20240405-1-b4fxau_html_9afa73172a43c75e.png) |
| **HOL02-Jupyter**| ![](RackMultipart20240405-1-b4fxau_html_abc3fcf89557a0a5.png) |

_Figure 4. Summary of security groups for each of the instances according to the configuration given by the instructor._

# Creation of the S3 bucket

The creation of the bucket is straightforward and most of the fields are left as default. This screenshot shows the bucket with a unique name (hol02-notebooks-sam).

![](RackMultipart20240405-1-b4fxau_html_fe453e2912ab1cab.png)

_Figure 5. My buckets._

# OpenVPN Configuration

Installing OpenVPN is easy through the command line. This subnet is easy to connect due to the flexibility of its security group.

![](RackMultipart20240405-1-b4fxau_html_9e45d86c253b2f52.jpg)

_Figure 6. After the installation, the output provides you with the user and password to set up OpenVPN._

Once installed, one has to connect through the browser to the admin version. There, some configuration has to be applied:

![](RackMultipart20240405-1-b4fxau_html_cbce032dba318d7c.png)

_Figure 7. Settings to be applied inside the admin window of OpenVPN to ensure connectivity._

To download the profile that is going to be used by the client in my host machine (Windows OS), I had to connect to the user version of OpenVPN (no admin slash), download and import the profile, and then connect to the VPN. It was straightforward and easy.

There is one nuisance in the configuration of this OpenVPN, and it is that **the Network Settings** had to be updated each time the IP of the instance changes. This meant having to import again and again the new user-locked profile to connect to the VPN.

![](RackMultipart20240405-1-b4fxau_html_b73b714afa687bf0.png)

_Figure 8. Network settings. The hostname changed when the AWS session ended._

![](RackMultipart20240405-1-b4fxau_html_287fa1152499104d.png)

_Figure 9. Screenshot of the OpenVPN Client for my host machine. Connection was successfully established. After this connection, I was able to connect to the EC2 Research instance and the Voila EC2 through SSH; otherwise, the connection could not be established._

# Setting up the Research EC2 instance

Because I created the instances from scratch using the settings provided in the instructions, once created I was unable to grant this instances access to the internet to update packages through yum or install packages. See [Troubleshooting](#Troubleshooting) section for details.

Once solved, I could successfully connect to the instance through the private IP and port 8888, if and only if I was attached to the VPN:

![](RackMultipart20240405-1-b4fxau_html_8c7d177962a044ce.png)

_Figure 10. Jupyter Notebook on the Research EC2 instance. Mind the connection was made through the private (and not the public IP) and the port is the default 8888._

# Setting up the Production instance

The installation of the Voila server is straightforward. The welcome screen is shown in the screenshot.

![](RackMultipart20240405-1-b4fxau_html_d782df303e79e67c.jpg)

_Figure 11. Nginx welcome screen after installation. Access is granted if one directly uses de IP._

Connection this instance to the S3 bucket is much easier, as there is an internet gateway. After installation of the keys, it can be easily accessed.

![](RackMultipart20240405-1-b4fxau_html_b706197f3b07f647.png)

_Figure 12. Command Line showing that the contents of the S3 bucket can be listed from the Production Subnet after configuring aws._

Because this instance runs Amazon Linux and it does not have cron as default to synchronize the notebooks with the S3, I used 'systemd' as a workaround (see [Troubleshooting](#Troubleshooting)).

# Syncing

Because cron jobs have been deprecated in Amazon Linux 2023, the syncing strategy has been explained in the [Troubleshooting](#Troubleshooting) section. Briefly, the recommended systemd approach has been implemented.

To test whether syncing works, I am first going to create a sample notebook on the Jupyter EC2 instance through the Jupyter Notebook interface.

![](RackMultipart20240405-1-b4fxau_html_8460b2103289374a.png)

_Figure 13. Creation of a file in the Jupyter environment on the Research instance._

We can see that this file does not exist yet on the S3 bucket as seen from the AWS control panel.

![](RackMultipart20240405-1-b4fxau_html_72f4632aea4ddea.png)

_Figure 14. The locally created file is not yet uploaded to the S3 bucket._

When the set timer triggers the syncing command, then this file created 'locally' on the Research instance must appear in the bucket seen from the AWS Control Panel. We wait for a few minutes and we check that the file has been successfully uploaded.

![](RackMultipart20240405-1-b4fxau_html_611273f419433b9b.png)

_Figure 15. In the AWS page we can see that the newly created notebook has been pushed to the bucket successfully._

Now it is time to check whether the syncing works in the Voila server instance. We are going to use the same systemd approach as in the Research EC2 instance.

![](RackMultipart20240405-1-b4fxau_html_6c57d3ef435e858d.png)

_Figure 16. We can see that at the beginning no 'Fireflies.ipynb' file exists in the instance. We check how much is left for syncing and wait. After the wait, we check again whether the file exists... and it does! It has been successfully pulled from S3._

# Troubleshooting

# Google Chrome incompatibility issues with Vocareum

The first problem I faced was accessing the AWS Lab, because Google Chrome browser has compatibility issues. Being a Windows OS user, I turned to Microsoft Edge which worked seamlessly.

# Installing applications on the Research EC2 instance

Because this instance is 'isolated', meaning that it has no direct internet connection, I found it difficult to install the Jupyter notebook or even routine applications due to the lack of connectivity.

In my workflow, I created the security groups from scratch within the instance launcher, so when the Research instance was created, it only accepted connections from the VPN and could not receive connections from the internet for the installation of packages.

My workaround was to **temporarily** modify:

1. The security groups, so I temporarily used the same as the VPN.
2. The routing table, so that this subnet had temporarily connection to the internet gateway, and thus to the internet.

Once I got Jupyter installed, I returned the settings to their original values.

# Access to the S3 bucket from the Research EC2 instance

Similarly, because this instance is not connected to the internet, the instance can not reach the bucket. The way I solved this was by creating an **endpoint gateway** that allowed for connection from the EC2 Research instance to the S3 bucket.

![](RackMultipart20240405-1-b4fxau_html_eb2a7592dd9fd08.png)

_Figure 17. Configuration of the endpoint gateway in AWS._

![](RackMultipart20240405-1-b4fxau_html_da7caaa10ac1691f.png)

_Figure 18. After creating the endpoint gateway, the buckets are listed and can be reached from the EC2 in the private subnet._

# Amazon Linux 2023 does not support cron

The cron package is not installed by default in this Linux distribution, so an alternative solution had to be considered for bucket synchronization. I have used the 'systemd' timers in a similar way to cron jobs. I followed some of the instructions on [this blog](https://hungrymonkey.github.io/systemd-s3-backups-v1/).

In the case of the Research Instance, which has to upload the files to the S3 bucket once they are created, I made a .timer file at/etc/systemd/system/s3-sync.timer. This sets up a timer that performs the task every hour. This can be set to every half an hour, every five minutes or other time interval that better suits the user.

[Unit]

Description=Sync S3 Bucket to Local Folder

[Timer]

OnCalendar=\*-\*-\* \*:00:00

[Install]

WantedBy=timers.target

Then I created a Systemd Service Unit File (with a .service extension) to define the synchronization command.

[Unit]

Description=Sync S3 Bucket to Local Folder

[Service]

Type=oneshot

ExecStart=/usr/bin/aws s3 sync /home/ec2-user/notebooks s3://hol02-notebooks-sam

Finally, I enabled and started the timer. The snapshot shows that the timer is enabled and waiting for the next trigger (hour).

![](RackMultipart20240405-1-b4fxau_html_14dc1404dd0969ae.png)

_Figure 19. S3 syncing on the Research EC2 instance. Syncing was achieved configuring systemd._

I have a problem, however, with this approach. Remote transfer requires login credentials, that's why the service file had to be slightly modified to add the login details of AWS CLI. This is the resulting .service file:

[Unit]

Description=AWS IAM user Backup

[Service]

Environment=AWS\_ACCESS\_KEY\_ID=\<access-key\>

Environment=AWS\_SECRET\_ACCESS\_KEY=\<secret-access-key\>

Environment=AWS\_SESSION\_TOKEN=\<session-token\>

Type=oneshot

ExecStart=/usr/bin/aws s3 sync /home/ec2-user/notebooks/ s3://hol02-notebooks-sam

I changed the timer setting to execute the command every five minutes for developing purposes (checking if it works is faster).

The order of the source and destination paths in the aws s3 sync command can indeed affect the behavior of the synchronization. The first path is the source path, indicating the local directory or S3 bucket/prefix from which files should be copied. The second path is the destination path, indicating the local directory or S3 bucket/prefix to which files should be copied. This way, in the Research Instance we are pushing the files, and in the Voila Instance we are pulling or retrieving the files. This is why to sync from the bucket to the Voila instance, the same steps were taken and only the _ExecStart_ line of the service file was modified by reversing the paths ordering.

# The elastic IP changes when the AWS session is closed

When the 4 hours are up and the session in AWS is closed, the IP changes and some configuration must be performed again on the VPN to allow for the connection.

# OpenVPN access

https:// instead of http:// has to be used to enter OpenVPN in the browser.

# Installing cronie as an alternative to cron

Unfortunately, crontab is not available by default in Amazon Linux 2023 EC2 instances. I found [this](https://jainsaket-1994.medium.com/installing-crontab-on-amazon-linux-2023-ec2-98cf2708b171)blog post that provides a step-by-step workaround to successfully install crontab on AL 2023 through the alternative package 'cronie'.

![](RackMultipart20240405-1-b4fxau_html_ed3167d8f45f500b.png)

_Figure 20. Cronie package installation through yum._

This is an alternative to the systemd approach explained before, but because it has been deprecated I have preferred to use the recommended way through systemd. I have not further explored, for this reason, this deployment.
