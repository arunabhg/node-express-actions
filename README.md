# Node.js Express app deploy to AWS EC2 using S3 artifact - starter template

### Steps

1.  Create a new repo or clone this repo.
    <br /><br />
2.  Set up AWS EC2.<br />
    2.1 Open a new EC2 instance and create a new Key pair. Copy to the home directory.<br />
    2.2 Go to the Security Groups. Create a new Security Group. Add ports `22 for SSH` and `3000 for Custom TCP` in _Inbound Rules_. They should be open for **Anywhere/IPV4 (0.0.0.0)**.<br /><br />
3.  Create a role with **S3 full access** and **Admin access**.<br /><br />
4.  If not already created, add a new IAM user and grant it the above role. Give it _programmatic_ access so that it can access via AWS CLI. _(No login via IAM/password is required)_<br /><br />
5.  Open AWS S3 and create a bucket with a name you want to save your files in. For eg., _node-express-typescript-artifact_.<br /><br />
6.  Open a command prompt (Administrator) and configure AWS CLI for the IAM user. (give command - `aws configure` in the command prompt and enter _access key_, _secret key_ and _region_ for the IAM user using the AWS console).<br />
    To install AWS CLI - https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html<br /><br />
7.  If on **Windows**, use a tool like WinRar or 7Zip to create a project.zip file containing all the project folders except the _node_modules_, _dist_ and _.github_. On **MacOS** or **Linux**, use the command line to create a zip of the project file - `zip -r project.zip . -x node_modules* dist* .git*` <br />
    _**Note -**_ Remember to NOT include the parent folder while creating the archive.<br /><br />
8.  Upload the zip file to S3 using the command - `aws s3 cp project.zip s3://<S3_bucket_name>/<folder_name>/project.zip`. <br />
    _**Note -**_ Remember to change the S3 bucket name and the underlying folder with the name and folder you have created the S3 bucket. For eg., `aws s3 cp project.zip s3://node-express-typescript-artifact/code-deploy-1/project.zip`.<br /><br />
9.  Create the EC2 instance and choose key pair and security group.<br /><br />
10. SSH into the instance using a SSH tool like Putty on Windows. On MacOS or Linux, you can use the terminal.<br />
    _**Note -**_ If you get an error like 'server refused our key' give appropriate user name for the AMI. If chosen **Ubuntu** AMI, the username will be `ubuntu` and for **AWS Linux** will be `ec2-user`. For the full troubleshooting tips for Putty - https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.html#TroubleshootingInstancesConnectingPuTTY<br /><br />
11. After logging into the server with the help of Putty, create a directory in it to put the files in.<br /><br />
12. Install and _configure_ AWS CLI in the instance _(inside server)_ as per the instance configuration.<br /><br />
13. Copy the project files in S3 bucket where we have stored our zip files to our EC2 instance using SSH. Use the URI mentioned in the S3 bucket to copy.<br /> _**Note -**_ Create an _EC2 role_ with S3 access first, and add the role to the instance. (You will need to modify the IAM role by clicking on the Actions of the EC2 instance. This will grant the EC2 instance access to S3.)<br /><br />
14. Unzip the zipped file copied from S3 bucket to the folder created.<br /> Command - `unzip -o project.zip -d Code/node-express-codedeploy1/`<br /><br />
15. Install Node.js in the server.<br /><br />
16. Run `npm run build`, if it's a TypeScript project and `npm run start`, if a node express one. Grab the IP from the EC2 instance and check on port 3000. You should have the same output as the local. Your backend Node.js is successfully deployed on EC2.<br /><br />
17. Add a systemd service on the server so that we don't always have to run the server manually. <br />The command - `sudo vim /etc/systemd/system/node-api.service`<br />
    17.1 Add the following code snippet and terminate with :wq! -<br />

        ```
             [Unit]
             Description=Nodejs Hello world App
             Documentation=https://example.com
             After=network.target

             [Service]
             Type=simple
             User=ubuntu
             ExecStart=/usr/bin/node /home/ubuntu/Code/node-express-codedeploy1/dist/index.js
             Restart=on-failure

             [Install]
             WantedBy=multi-user.target
        ```

    17.2 Reload the service - `sudo systemctl daemon-reload`<br />
    17.3 Enable the service - `sudo systemctl enable node-api.service`<br />
    17.4 Start the service - `sudo systemctl start node-api.service`<br />
    17.5 View the service journal - `sudo journalctl -fu node-api.service`<br />
    After you give all the above commands you will see that your app is running on server and you don't have to start it every time.

## When you make any changes in code on local, run the following commands to upload, build and propagate the changes on the server

## In the local terminal -

Follow Steps 7 & 8 to create & upload a new zip to S3 bucket.

## In the server -

`aws s3 cp s3://node-express-typescript-artifact/code-deploy-1/project.zip project.zip unzip -o project.zip -d Code/node-express-codedeploy1/ npm install --prefix Code/node-express-codedeploy1/ npm run build --prefix Code/node-express-codedeploy1/ sudo systemctl restart node-api.service`

---

**_Note -_** This template doesn't use CI/CD to deploy code to EC2. We have to run a command manually using Putty or any other SSH client, each time we make some changes, to push those changes to the server.
