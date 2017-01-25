# CDH: Installation Path A
This repository contains the steps I took to install CDH on Amazon EC2 instances using Install Path A

1. Logged into my AWS account
2. Locked into U.S. East (N. Virginia)
3. Created a public/private key pair according to these instructions: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
4. Security group creation should be done before the instance configuration.  See the setup I have for the default group which allows the inbound ports specified for the Cloudera installation.

<pre>
