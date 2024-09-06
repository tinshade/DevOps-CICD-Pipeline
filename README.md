#  DevOps Essentials Lab Cheat Sheet
Welcome to the DevOps Essentials Lab Cheat Sheet!

![how-devops](https://github.com/janjiralakirankumar/DevOpsEssentials/assets/137407373/b0493ccd-3370-4f98-b08a-6e76626ca226)

This guide provides step-by-step instructions for completing various DevOps labs, covering tasks like: 
`Setting up servers using Terraform,` `Working with Git and GitHub,` `Configuring Jenkins and GitWebHook`.

### DevOps Essentials Lab Pre-requisites
1. Basic understanding of Linux commands,
2. Basic knowledge of a Cloud platform such as AWS,
3. It's good to have an AWS-Free Tier Account for Practice.
---
## Lab 1: Use Terraform and Ansible to set up `Jenkins Server` for CICD Lab.

**Objective:**
The objective of this lab is to set up an AWS EC2 instances for Jenkins using Terraform. 
This lab aims to provide a foundation for building a Continuous Integration/Continuous Deployment (CICD) environment.

---------------------------------------------------------------------
### Task-0: The first step is to `Manually Launch an EC2 Server` with the below configuration:

* **Region:** North Virginia (us-east-1).
* **Use tag Name:** `CICDLab-YourName`
* **AMI Type and OS Version:** `Ubuntu 22.04 LTS`
* **Instance type:** `t2.micro`
* Create a new Keypair with the Name `CICDLab-Keypair-YourName`
* In security groups, include ports `22 (SSH)` and `80 (HTTP)` (Add remaining ports later)
* **Configure Storage:** 10 GiB
* Click on `Launch Instance.`
* Once it is Launched, Ensure to add the remaining ports in the security group ie... `8080,` `9999,` and `4243.`

---------------------------------------------------------------------
### Task-1: Installing Terraform onto Anchor Server.

Once the Anchor EC2 server is up and running, SSH into the machine using `MobaXterm` or `Putty` with the username `ubuntu` and do the following:
```
sudo hostnamectl set-hostname CICDLab
bash
```
```
sudo apt update
```
```
sudo apt install wget unzip -y
```
```
wget https://releases.hashicorp.com/terraform/1.9.4/terraform_1.9.4_linux_amd64.zip
```
View the [Terraform's Latest Versions](https://developer.hashicorp.com/terraform/downloads)
```
unzip terraform_1.9.4_linux_amd64.zip
ls
```
```
sudo mv terraform /usr/local/bin
```
```
ls
terraform -v
```
```
rm terraform_1.9.4_linux_amd64.zip
```
---------------------------------------------------------------------
### Task-2: Install Python 3, pip, AWS CLI, and Ansible
Install Python 3 and the required packages:
```
sudo apt-get update
sudo apt-get install python3-pip -y
```
```
sudo pip3 install awscli boto boto3 ansible
```
```
aws configure
```
#### Enter the Credentials 
---------------------------------------------------------------------
**Note:** If you want to create new credentials, Follow the below steps:

1. Go to the AWS console. On the top right corner, click on your name or AWS profile ID.
2. Click on Security Credentials.
3. Under AWS IAM Credentials, click on **Create Access Key**.
4. If you already have two active keys, you can deactivate and delete the older one so that you can create a new one, then download, and save it.
5. Then, Complete the `aws configure` step

---------------------------------------------------------------------
#### Once configured, do a smoke test to check if your credentials are valid and get access to AWS.

You can check using any one command
```
aws s3 ls
```
(Or)
```
aws iam list-users
```

#### Create a host inventory file with the necessary permissions
 ```
 sudo mkdir /etc/ansible && sudo touch /etc/ansible/hosts
```
```
 sudo chmod 766 /etc/ansible/hosts
```
**Note:** The above command gives `read and write permissions to the owner,` `read and write permissions to the group,` and `read and write permissions to others.`

---------------------------------------------------------------------
### Task-3: Use Terraform to launch jenkins server
Create the Terraform configuration and variables files as described.
* We need to create one jenkins servers
* For **Git Operations** we will use the same **Anchor EC2** from where we are operating now 

#### Create the terraform directory and set up the config files in it
```
mkdir devops-labs && cd devops-labs
```
As a first step, create a key using ssh-keygen.
```
ssh-keygen -t rsa -b 2048 
```
##### Explanation:

* `-t rsa:` Specifies the type of key to create, in this case, RSA.
* `-b 2048:` Specifies the number of bits in the key, 2048 bits in this case. The larger the number of bits, the stronger the key.

**Note:**
1. This will create `id_rsa` and `id_rsa.pub` in **/home/ubuntu/.ssh/**
2. Keep the path as **/home/ubuntu/.ssh/id_rsa**; don't set up any passphrase, just hit the '**Enter**' key for the 3 questions it asks

#### Now create the Terraform config files.
```
vi DevOpsServers.tf
```
Copy and paste the below code into `DevOpsServers.tf`
```
provider "aws" {
  region = var.region
}

resource "aws_key_pair" "mykeypair" {
  key_name   = var.key_name
  public_key = file(var.public_key)
}

# to create 2 EC2 instances
resource "aws_instance" "my-machine" {
  ami                    = var.ami_id
  key_name               = var.key_name
  vpc_security_group_ids = [var.sg_id]
  instance_type          = var.ins_type
  tags = {
    Name = "jenkins-server"
  }

  provisioner "local-exec" {
    command = <<-EOT
      echo ${self.public_ip} >> /etc/ansible/hosts
    EOT
  }
}
```
Now, create the variables file with all variables to be used in the main config file.
```
vi variables.tf
```
Change the following data in variables.tf File. 
1. Edit the **Allocated Region** (**Ex:** ap-south-1) & **AMI ID** of same region,
2. Replace the same **Security Group ID** Created for the Anchor Server
3. Add your Name for **KeyPair** (Example: "**YourName**-CICDlab-KeyPair")

```
variable "region" {
    default = "us-east-1"
}

# Change the SG ID. You can use the same SG ID used for your CICD anchor server
# Basically the SG should open ports 22, 80, 8080, 9999, and 4243
variable  "sg_id" {
    default = "sg-06dc8863d3ed3d280" # us-east-1
}

# Choose a free tier Ubuntu AMI. You can use below. 
variable "ami_id" {
    default = "ami-04505e74c0741db8d" # us-east-1; Ubuntu
}

# We are only using t2.micro for this lab
variable "ins_type" {
    default = "t2.micro"
}

# Replace 'yourname' with your first name
variable key_name {
    default = "YourName-CICDlab-KeyPair"
}

variable public_key {
    default = "/home/ubuntu/.ssh/id_rsa.pub"   #Ubuntu OS
}

```
#### Now, execute the terraform config files to launch the servers
```
terraform init
```
```
terraform fmt
```
```
terraform validate
```
```
terraform plan
```
```
terraform apply -auto-approve
```
#### After the terraform code is executed, check the host's inventory file and ensure the below output.
```
cat /etc/ansible/hosts
```

* [jenkins-server] 44.202.164.153

**(Optional Step):** When you stop and start the EC2 Instances Public IP Changes. In that case, To update Jenkins Public IP addresses use the below command
```
sudo vi /etc/ansible/hosts 
```
Once Updated, Save the File.

#### Now `SSH` into `Jenkins-server` and check they are accessible from `Anchor EC2`
```
ssh ubuntu@<Jenkins ip address>
```
#### Set the hostname as
```
sudo hostnamectl set-hostname Jenkins
bash
```
```
sudo apt update
```
**Exit** only from the Jenkins Server, not the Anchor Server.

---------------------------------------------------------------------
### Task-4: Use Ansible to deploy respective packages onto each of the servers 

#### Create a directory and change to it
```
cd ~
mkdir ansible && cd ansible
```
#### Now, Download the playbook, which will deploy packages onto the `Jenkins-Server.`
```
wget https://raw.githubusercontent.com/comal21/DevOps-CICD-Pipeline/main/DevOpsSetup.yml
```
#### Run the above playbook to deploy the packages
```
ansible-playbook DevOpsSetup.yml
```
#### At the end of this step, `Jenkins-Server` will be ready for performing further Labs

#### Verify that the Jenkins landing page is working.
* Use your respective Jenkin'sPublic IP address as shown: http://**44.202.164.153**:8080/ 

---------------------------------------------------------------------
**Summary:**
1. Launch an EC2 instances in AWS for jenkins.
2. Install Terraform on the Jenkins server to automate infrastructure provisioning.
3. Configure AWS CLI and Ansible for managing resources.
4. Create a Terraform configuration to define the servers and their attributes.
5. Launch the servers using Terraform.
6. Update the Ansible hosts file with the server details.
7. Configure Jenkins server with proper host names.
8. Use Ansible to install necessary software packages and dependencies on the server.

#### =============================END of LAB-01=============================
---
## Lab 2: Git and GitHub Operations.

**Objective:**
This lab focuses on `Git` and `GitHub operations,` including `initializing a Git repository,` `making commits,` `creating branches,` and `pushing code to a GitHub repository.`

### Pre-requisites:
1. Create a **GitHub Account** & **Empty Public Repository** with name as **"hello-world"**

   **Note:** (To SignUp for GitHub Account, [Click Here](https://github.com/signup?ref_cta=Sign+up&ref_loc=header+logged+out&ref_page=%2F&source=header-home))
2. You need to generate a **Personal Access Token** (PAT) in GitHub.
3. Basic familiarity with **Git Commands.**

Once the GitHub Account and Empty repository is ready, let's operate in the local Anchor Server.

---------------------------------------------------------------------
### Task-1: Initializing the local git repository and committing changes

On the `CICD anchor EC2,` do the below:
```
cd ~/
```
Check the GIT version. 
```
git --version
```
(Optional) If it does not exist, then you can install it with the below command. Else no need to execute the below line:  
```
sudo apt install git -y
```
Download the **Java Code** that we are going to use in the CICD pipeline.
```
wget https://hsbctraining.s3.us-east-2.amazonaws.com/hello-world-master.zip
```
```
unzip hello-world-master.zip
```
```
ls
rm hello-world-master.zip
```
```
cd hello-world-master
ls
```
```
git init .
```
To set the `User Identity` ie... `email and user name.` you can use below:

```
git config --global user.email "< Add your email >"
```
```
git config --global user.name "< Add your Name >"
```
```
git status
```
```
git add .
```
```
git status
```
```
git log
```
```
git commit -m "This is the first commit"
```
```
git log
```
```
git status
```

---------------------------------------------------------------------
### Task-2: Pushing the Code to your Remote GitHub Repository  

* To push code to **Github**, You need to generate a **Personal Access Token** (PAT) in github.

**To Generate the Token Follow the below steps:**

* First, Go to your `GitHub homepage,` Click on the top right `profile Icon` and then `settings`
* Click on `Developer settings` (At the bottom on the left side menu). Click on `Personal Access Token` and then Click on `Generate New Token.`
* Under '**Select Scopes**' select all items. Click on '**generate token**'.
* Once generated, **Copy and save the token in a safe location as it is not visible again in GitHub.**


* Create `Alias` as `Origin` to GitHub's Remote repository URL.
```
git remote add origin < Replace your Repository URL > 
```
(**Example:** `git remote add origin https://github.com/janjiralakirankumar/hello-world.git`)

* To view the list of Aliases, run the below command in Terminal.
```
alias
```
* To view a specific alias use the below command.
```
git remote show origin
```
* Now you can push your code to the Remote repository using the below command.
```
git push origin master 
```
* When it asks for a password, enter the **Personal Access Token** and Press Enter (PAT is invisible and It's the expected behavior.)
---------------------------------------------------------------------
**Summary:**
1. Initialize a local Git repository on Anchor EC2 instance.
2. Configure Git settings like email and username.
3. Add and commit changes to the Git repository.
4. Push the code to a GitHub repository.

#### =============================END of LAB-02=============================
---
## Lab 3: Configuring Jenkins server and Installing Tomcat onto Jenkin's Server for Deploying our Application.

**Objective:**
The objective of this lab is to configure Jenkins to build and deploy applications. It includes `Setting up Jenkins,` `installing necessary plugins` and `configuring Jenkins to build Maven projects,` and `Installing Tomcat Server.` 

---------------------------------------------------------------------
### Task-1: Configure Jenkins Server:

Using Jenkin's `Public IP` SSH into the **Jenkins Server** from the Anchor Server.
```
ssh ubuntu@xx.xx.xx.xx
```
Get the **Initial Password** for Jenkins from the below path.
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
   (**Example:** "afbe8d33e25b4b908c0b9f91546f09e6")

1. Now, go to the **Web Browser** and enter the Jenkins URL as shown: **http://< Jenkin's Public IP>:8080/**
2. Under Unlock Jenkins, enter the above **initialAdminPassword** & click **Continue**.
3. Click on **Install suggested Plugins** on the Customize Jenkins page.
4. Once the plugins are installed, it gives you the page where you can create a New **Admin User**. 
5. Enter the **User Id** and **Password** followed by **Name and E-Mail ID** then click on **Save & Continue**.
6. In the next step, on the Instance Configuration Page, verify your **Jenkins Public IP** and **Port Number** then click on **Save and Finish**

#### Now you will be prompted to the Jenkins Home Page

1. Click on **Manage Jenkins** > **Plugins** > **Available Plugins** tab and search for **Maven**.
2. Select the "**Maven Integration**" Plugin and "**Unleash Maven**" Plugin and click **Install** (without restart).
3. Once the installation is completed, click on **Go back to the top page**.
4. On Home Page select **Manage Jenkins** > **Tool**.
5. Inside Tool Configuration, look for **Maven installations**, click **Add Maven**. 
6. Give the **Name as "Maven"**, choose **Version as 3.9.5**, and **Save** the configuration.
7. Now you need to make a project for your application build, that selects **New Item** from the Home Page of Jenkins
8. Enter an item name as **hello-world** and select the project as **Maven Project** and then **click OK.**
   ( You will be prompted to the configure page inside the hello-world project.)
9. Go to the "**Source Code Management**" tab, and select Source Code Management as **Git**, Now you need to provide the GitHub Repository **Master Branch URL** and **GitHub Account Credentials**.
10. In the Credentials field, you have to click **Add** and then click on **Jenkins**.
11. Then you will be prompted to the **Jenkins Credentials Provider** page. Under Add Credentials, you can add your **GitHub Username**, **Password**, and **Description**. Then click on **Add**.
12. After returning to the Source Code Management Page, click on Credentials and Choose your **GitHub Credentials**.
13. Keep all the other values as default and select the "**Build**" Tab and inside Goals and options write "**clean package**" and **save** the configuration.

#### Note: The 'clean package' command clears the target directory, Builds the project, and packages the resulting WAR file into the target directory.

16. Now, you will get back to the Maven project "**hello-world**" and click on "**Build Now**" to build the **.war** file for your application

* You can go to **Workspace** > **dist** folder to see that the **.war** file is created there.
* war file will be created in **/var/lib/jenkins/workspace/hello-world/target/**

---------------------------------------------------------------------
### Task-2: Installing and Configuring Tomcat for Deploying our Application on Jenkins Server

![image](https://github.com/janjiralakirankumar/DevOpsEssentials/assets/137407373/d5dde194-f10d-4b4d-a20c-890e9ca3e392)

* Now, SSH into the Jenkins server (Make sure that you are the root user and Install the Tomcat web server)
* **Note:** (If you are already in Jenkins Server, again SSH is not needed.)

1. Follow below steps:
```
sudo apt update
```
```
sudo apt install tomcat10 tomcat10-admin -y
```
```
sudo systemctl enable tomcat10
```
Now we need to navigate to **server.xml** to change the Tomcat port number from **8080 to 9999**.
(As port number 8080 is already being used by the Jenkins website)
```
sudo vi /etc/tomcat10/server.xml
```
**(Optional step):** If you are unable to open the file then change the permissions by using the below command.
```
sudo chmod 766 /etc/tomcat10/server.xml
```
#### Change 8080 to 9999
* press esc & Enter **":"** and copy paste below code and hit enter
```
g/8080/s//9999/g
```
Save the file using `ESCAPE+:wq!`

* To Verify whether the Port is changed, execute the below Command.
```
cat /etc/tomcat10/server.xml
```
**(Optional step):** If you are unable to open the file then change the permissions by using the below command.
```
sudo chmod 766 /etc/tomcat10/server.xml
```
Now restart the system for the changes to take effect
```
sudo service tomcat10 restart
```
```
sudo service tomcat10 status
```
**To exit**, press **ctrl+c**

* Once the Tomcat service restart is successful, go to your web browser and enter **Jenkins Server's Public IP address** followed by **9999** port.

(Example: **http://< Jenkins Public IP >:9999**     or     **http://184.72.112.155:9999**)

* Now you can check the Tomcat running on **port 9999** on the same machine.
* We need to copy the **.war** file created in the previous Jenkins build from the Jenkins workspace to tomcat webapps directory to serve the web content
```
sudo cp -R /var/lib/jenkins/workspace/hello-world/target/hello-world-war-1.0.0.war /var/lib/tomcat10/webapps
```
The above command is copying a `WAR (Web Application Archive)` file from the Jenkins workspace to the Tomcat web apps directory. Let's break down the command:

- `sudo`: Run the command with superuser (root) privileges, as copying files to system directories often requires elevated permissions.

- `cp`: The copy command in Linux.

- `-R`: Recursive option, used when copying directories. It ensures that the entire directory structure is copied.

- `/var/lib/jenkins/workspace/hello-world/target/hello-world-war-1.0.0.war`: Source path, specifying the location of the WAR file in the Jenkins workspace.

- `/var/lib/tomcat9/webapps`: Destination path, indicating the Tomcat webapps directory where the WAR file is being copied.

This command assumes that your Jenkins job has built a WAR file named `hello-world-war-1.0.0.war` in the specified workspace directory. It then copies this WAR file into the Tomcat webapps directory, allowing Tomcat to deploy and run the web application.

* Once this is done, go to your browser and enter Jenkins Public IP address followed by port 9999 and path (URL:  **http://< Jenkins Public IP >:9999/hello-world-war-1.0.0/**).
* Now, you can see that Tomcat is now serving your web page
* Now, Stop tomcat9 and remove it. Otherwise, it will slow down the Jenkins server.
```
sudo service tomcat10 stop
```
```
sudo apt remove tomcat10 -y
```
---------------------------------------------------------------------

## Lab 4: Using GitWebHook to build your code automatically using Jenkins

**Objective:**
This lab focuses on configuring Git WebHooks to automatically trigger Jenkins builds when code changes are pushed to a GitHub repository.

---------------------------------------------------------------------
### Task-1: Configure Git WebHook in Jenkins

1. Go to the Jenkins webpage and choose **Manage Jenkins** > **Plugins**
2. Go to the **Available** Tab, Search for **GitHub Integration**. Select the **GitHub Integration Plugin** and then on **Install** (without restart).
3. Once the installation is complete, click on **Go back to the top page**.
4. In your **hello-world project**, Click on **Configure**.
5. Go to **Build Triggers** and enable the **GitHub hook trigger for GITScm polling**. Then **Save**.
6. Go to your **GitHub website**, and inside the **hello-world** repository > **Settings** > **Webhooks** and Click on the **Add Webhook**.
7. Now fill in the details as below.
#### Payload URL Example: 
* http://< jenkins-PublicIP >:8080/github-webhook/ (**Example:** http://184.72.112.155:8080/github-webhook/)
* **Content type:** application/JSON

Then, Click on **Add Webhook**.

----------------------------------------------------------------
### Task-2: Verifying whether the WebHook is working or not by editing the Source Code

1. Now, Make a minor change and commit in GitHub's **hello-world repository** by editing **hello.txt** file.
2. As the source code gets changed, Jenkins gets triggered by the WebHook and starts building the new source code.
3. Go to Jenkins, and you can see a build is happening automatically.
4. Observe the successful build on the Jenkins page.
---------------------------------------------------

**Summary:**
1. Configure Git WebHooks in Jenkins for automatic triggering of builds.
2. Create a GitHub WebHook for a specific GitHub repository.
3. Make a minor code change in the GitHub repository to trigger a build in Jenkins.
4. Verify that Jenkins successfully starts a new build when changes are pushed.

#### =============================END of LAB-04=============================


**Summary:**
1. Opening the `Jenkins Web Page` on the browser.
2. Unlock Jenkins and create an admin user.
3. Install plugins, including Maven integration.
4. Configure Jenkins to use a specific version of Maven.
5. Create a Maven project in Jenkins for the "hello-world" application.
6. Configure source code management with Git and GitHub.
7. Define build goals and options.
8. Build the project and verify the outcome.
9. Install and configure Apache Tomcat for serving web applications.
10. Deploy the built WAR file to Tomcat.

#### =============================END of LAB-03=============================

## Important Git Commands

Here are some important Git commands and their brief explanations:

1. **`git init`**
   - Initializes a new Git repository in the current directory.

   ```bash
   git init
   ```

2. **`git clone`**
   - Copies a repository from a remote source (like GitHub) to your local machine.

   ```bash
   git clone <repository_url>
   ```

3. **`git add`**
   - Adds changes in the working directory to the staging area for the next commit.

   ```bash
   git add <file>
   ```

4. **`git commit`**
   - Records changes in the repository with a message describing the changes.

   ```bash
   git commit -m "Your commit message"
   ```

5. **`git status`**
   - Shows the status of changes as untracked, modified, or staged.

   ```bash
   git status
   ```

6. **`git diff`**
   - Shows the differences between the working directory and the latest commit.

   ```bash
   git diff
   ```

7. **`git log`**
   - Displays a log of commits, showing commit messages, authors, dates, and commit hashes.

   ```bash
   git log
   ```

8. **`git branch`**
   - Lists existing branches or creates a new branch.

   ```bash
   git branch          # List branches
   git branch <branch_name>   # Create a new branch
   ```

9. **`git checkout`**
   - Switches to a different branch or restores working files from a specific commit.

   ```bash
   git checkout <branch_name>   # Switch to a branch
   git checkout -b <new_branch>   # Create and switch to a new branch
   ```

10. **`git merge`**
    - Combines changes from different branches.

    ```bash
    git merge <branch_name>
    ```

11. **`git pull`**
    - Fetches changes from a remote repository and integrates them into the current branch.

    ```bash
    git pull origin <branch_name>
    ```

12. **`git push`**
    - Uploads local branch commits to the remote repository.

    ```bash
    git push origin <branch_name>
    ```

13. **`git remote`**
    - Lists remote repositories that are connected to your local repository.

    ```bash
    git remote -v   # Show remote repositories with URLs
    ```

14. **`git fetch`**
    - Downloads changes from a remote repository without merging them into your working branch.

    ```bash
    git fetch origin
    ```

15. **`git reset`**
    - Resets the current branch to a specific commit, optionally preserving changes in the working directory.

    ```bash
    git reset --hard <commit_hash>
    ```

These are just a few fundamental Git commands. There are many more, each serving different purposes in the version control process.

---
## DevOps Tools and Documentation Links:

Certainly! Here are the official websites and documentation links for some popular DevOps tools:

1. **Version Control:**
   - **Git:**
     - Website: [Git](https://git-scm.com/)
     - Documentation: [Git Documentation](https://git-scm.com/doc)

2. **Continuous Integration and Continuous Deployment:**
   - **Jenkins:**
     - Website: [Jenkins](https://www.jenkins.io/)
     - Documentation: [Jenkins Documentation](https://www.jenkins.io/doc/)
   - **Travis CI:**
     - Website: [Travis CI](https://travis-ci.com/)
     - Documentation: [Travis CI Documentation](https://docs.travis-ci.com/)

3. **Configuration Management:**
   - **Ansible:**
     - Website: [Ansible](https://www.ansible.com/)
     - Documentation: [Ansible Documentation](https://docs.ansible.com/)
   - **Puppet:**
     - Website: [Puppet](https://puppet.com/)
     - Documentation: [Puppet Documentation](https://puppet.com/docs/)

4. **Containerization and Orchestration:**
   - **Docker:**
     - Website: [Docker](https://www.docker.com/)
     - Documentation: [Docker Documentation](https://docs.docker.com/)
   - **Kubernetes:**
     - Website: [Kubernetes](https://kubernetes.io/)
     - Documentation: [Kubernetes Documentation](https://kubernetes.io/docs/)

5. **Infrastructure as Code:**
   - **Terraform:**
     - Website: [Terraform](https://www.terraform.io/)
     - Documentation: [Terraform Documentation](https://www.terraform.io/docs/index.html)

6. **Monitoring and Logging:**
   - **Prometheus:**
     - Website: [Prometheus](https://prometheus.io/)
     - Documentation: [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
   - **ELK Stack (Elasticsearch, Logstash, Kibana):**
     - Website: [Elastic](https://www.elastic.co/)
     - Documentation: [Elastic Stack Documentation](https://www.elastic.co/guide/index.html)

Always check the official websites for the most up-to-date information and documentation. Additionally, many of these tools may have vibrant communities where you can find additional resources, tutorials, and support.

#### ============================= END OF All LABs =============================

