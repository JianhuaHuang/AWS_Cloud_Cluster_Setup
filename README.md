-   [Set up R & Rstudio Working Environment on AWS Clusters](#set-up-r-rstudio-working-environment-on-aws-clusters)
    -   [1. Creat AMI for master node.](#creat-ami-for-master-node.)
        -   [Set Up Base Starcluster AMI Instance](#set-up-base-starcluster-ami-instance)
        -   [Connect to the instance via SecureCRT](#connect-to-the-instance-via-securecrt)
        -   [Install R](#install-r)
        -   [Install Rstudio Server](#install-rstudio-server)
        -   [Install other software such as Energyplus](#install-other-software-such-as-energyplus)
        -   [Create AMI](#create-ami)
    -   [2. Create AMI for slave nodes](#create-ami-for-slave-nodes)
    -   [3. Create Instance to Host `starcluster`](#create-instance-to-host-starcluster)
    -   [4. Set Up the Cluster](#set-up-the-cluster)
    -   [5. Upload Data](#upload-data)
    -   [6. Errors and FAQs](#errors-and-faqs)
    -   [7. Version Control on Git](#version-control-on-git)

Set up R & Rstudio Working Environment on AWS Clusters
======================================================

This document shows the processes of building, configuring, and managing AWS clusters based on *StarCluster*. StarCluster is an open source cluster-computing toolkit for Amazon EC2 (Elastic Compute Cloud). It comes with some publicly available AMIs (Amazon Machine Image), which include the software needed for parallel computing and distribution pre-installed. In this document, we should launch a base starcluster AMI, customize the AMI by including R, Rstudio, and EnergyPlus, create a new AMI, and use it to build the cluster.

1. Creat AMI for master node.
-----------------------------

### Set Up Base Starcluster AMI Instance

A base starcluster instance is needed to setup the image for the master node.

1.  To launch an instance, click **EC2** on the AWS console, click **instance** on the left panel, and then click **Launch Instance**.
2.  Search *starcluster* in the Community AMIs, and select the *starcluster-base-ubuntu-12.04-x86\_64 - ami-765b3e1f* image. The Ubuntu-12.04 is used (instead of the latest Ubuntu-13.04) because there are some errors when installing R and Rstudio on the Ubuntu-13.04 image.
3.  choose *m3.medium* as instance type in the step of **Choose Instance Type**.
4.  Then click the **Next** button until the step of **Configure Security Group**. At this step, click the **Add Rule** button, and add *HTTP* (port 80), *HTTPs* (port 443), and *Custom TCP Rule* (port 8787 for Rstudio) to the **Type** option, so that we can connect to the instance by browser later. (This step seems to be optional, since we can add the permissions to the starcluster config file later)
5.  Everything is ready now. Just click the **Launch** button.
6.  In the last step, create a new key pair named *starclusterMasterAMI*, download and save it to the local disk, which is needed for SSH connection later. Now, click **Launch Instances** to launch it.

### Connect to the instance via SecureCRT

After setting up the base starcluster instance, we can now connect to it using SecureCRT following this link: <http://blog.skufel.net/2012/09/how-to-connect-to-amazon-ec2-linux-ami-using-securecrt/>. The Hostname is the instance's public DNS, and Username is *ubuntu*. Use the downloaded *starclusterMasterAMI.pem* as Publickey for Authentication.

### Install R

Inside the SecureCRT terminal, we can now install the software we need for the master node. A simple way is available to install R in Ubuntu. Just use `sudo apt-get install r-base-core` to install the stable R version. But this R version is old (2.14) and a lot of new packages are not supported. Thus we are going to install the latest R version in a different way. Here is the reference link and steps (<http://stackoverflow.com/questions/16093331/how-to-install-r-version-3-0>):

1.  Adding deb to sources.list

        sudo vim /etc/apt/sources.list    
        deb http://cran.rstudio.com/bin/linux/ubuntu precise/  # add this line to sources.list

    precise is your Ubuntu name (may be different)

2.  Add key to sign CRAN packages

        sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E084DAB9

3.  Add specific PPA to the system

        sudo add-apt-repository ppa:marutter/rdev
        sudo apt-get update

4.  installing

        sudo apt-get install r-base

Type *R* in the terminal, and we can find the R version is 3.1.1 (2014-07-10).

### Install Rstudio Server

The installation of Rstudio is straightforward following this link: <http://www.rstudio.com/products/rstudio/download-server/>

    sudo apt-get install gdebi-core
    sudo apt-get install libapparmor1 # Required only for Ubuntu, not Debian
    wget https://download2.rstudio.org/rstudio-server-0.99.483-amd64.deb
    sudo gdebi rstudio-server-0.99.483-amd64.deb

After the installation finished, we should verify the installation is successful.

    sudo rstudio-server verify-installation

Now we can connect to the Rstudio through browser with the instance's public DNS, followed by *:8787*. When the website is opened, the rstudio-server will ask for user name and password. We should create them in the SecreCRT for use.

    sudo adduser jhuang

When it ask for *New Password*, enter the password you want (it is *rstudio* in this case). Then give the new user administrator right.

    sudo adduser jhuang sudo

Now, we can login to Rstudio with the user name and password

### Install other software such as Energyplus

For your own purpose, you may need to install other software. Here, I introduce how to install the EnergyPlus, which is widely used to simulate building energy consumption. Inside Rstudio console, we can upload the Energyplus software (SetEPlusV720006-lin-64.sh) to the instance using the Files--&gt;Upload button.

Then go back to SecureCRT to install Energyplus

    sudo sh /home/jhuang/SetEPlusV720006-lin-64.sh

use the default install directory. When it shows the following information, just click enter to go to the next step:

    EnergyPlus install directory [/usr/local]:

    Symbolic link location (enter "n" for no links) [/usr/local/bin]:

The installation requires a password, which is included in the downloading email. If the installation is successful, we should be able to run the help command

    sudo runenergyplus help

We can further test whether Energyplus works with building prototype file (idf) and weather data (epw). Create a new folder named test within */home/jhuang*, and upload the building.idf and weather.epw file to it.Now, we can test whether it works in SecureCRT.

    sudo runenergyplus /home/jhuang/test/MediumOfficeNew2004_1A_USA_FL_MIAMI.idf /home/jhuang/test/weather.epw 

If the simulation finish successfully, we can see an Output folder within /home/jhuang/test, and a lot of output files inside the output folder.

### Create AMI

After the successful installation and test of EnergyPlus, we should delete the test folder, and keep the instance a clean environment.

    sudo rm -R /home/jhuang/test

Then we can create an AMI for the instance in the AWS console. The processes of creating AMI is simple. Just go to AWS console, select **Instances -&gt; Actions -&gt; Create Image**. After a couple of minutes, an AMI will be available in the AMIs panel .

2. Create AMI for slave nodes
-----------------------------

The AMI for master node added R, Rstudio, and Energyplus to the base starcluster AMI. However, the slave nodes only need Energyplus to run the simulation. Thus, we can build a simpler AMI for the slave nodes with only Energyplus added to the base starcluster AMI. It may speed up the loading process of cluster setup. But this step is optional, and the slave nodes can also use the master AMI directly.

Because R and Rstudio are not installed in the slave AMI, we can't upload Energyplus to the instance via Rstudio console. Alternatively, we can upload it using FileZilla. To build the connection between FileZilla and EC2 instance, just follow this instruction: <http://stackoverflow.com/questions/16744863/connect-to-amazon-ec2-file-directory-using-filezilla-and-sftp>

3. Create Instance to Host `starcluster`
----------------------------------------

StarCluster is a toolkit used to control the Amazon cluster. With the StarCluster package, we can easily start the cluster, set the bidding price, add nodes to it, and so on. The StarCluster package is well documented in the website: <http://star.mit.edu/cluster/docs/0.92rc2/index.html>. In this section, we will build an instance to hold the starcluster package, and edit the configuration file following the Quick-Start Instruction: <http://star.mit.edu/cluster/docs/0.92rc2/quickstart.html>.

1.  Build a free Amazon Instance (*Amazon Linux AMI 2014.03.2 (HVM) - ami-76817c1e*) using the t2.micro instance type with default settings for each step. Create a new key pair for the instance, and save the key pair as *starclusterHost.pem* for SecureCRT connection.

2.  Connect to the starclusterHost through SecureCRT, and install the starcluster package

<!-- -->

    sudo easy_install StarCluster

1.  Create an EBS volume used to store the outputs. We can create the volume via the AWS console with **Volumes -&gt; Create Volume**, or in the terminal with:

<!-- -->

    starcluster createvolume --name=myvol 20 us-east-1a

The above command will launch a host instance in the us-east-1a availability zone, create a 20GB volume, attach the new volume to the host instance, and format the entire volume. The host instance will be t1.micro in default, so that it will not charge any money to host the volume. After we create the volume in either way, we can attach it to the master node later by editing the config file.

1.  Write the configuration template file

<!-- -->

    starcluster help
    StarCluster - (http://web.mit.edu/starcluster) (v. 0.9999)
    Software Tools for Academics and Researchers (STAR)
    Please submit bug reports to starcluster@mit.edu

    cli.py:87 - ERROR - config file /home/user/.starcluster/config does not exist

    Options:
    [1] Show the StarCluster config template
    [2] Write config template to /home/user/.starcluster/config
    [q] Quit

    Please enter your selection:

select **2** to write the config template to /home/user/.starcluster/config. Then copy the config to have a backup named config\_bk

    sudo cp /home/ec2-user/.starcluster/config /home/ec2-user/.starcluster/config_bk

1.  Edit the config file:

<!-- -->

    sudo vim ~/.starcluster/config

The following information is required to update

    [aws info]
    aws_access_key_id = #your aws access key id here
    aws_secret_access_key = #your secret aws access key here
    aws_user_id = #your 12-digit aws user id here

The access\_key\_id and secret\_accesskey are created in the AWS IAM console. The *User Name* is JianhuaHuang and *Policy Name* is *AdministratorAccess*. During the creation, the key information was saved to *C:\\Users\\temp\\Dropbox\\AWS\\JianhuaHuang.credentials.csv*. The 12-digits aw\_user\_id can be found in the AWS account. Update the \[AWS info\] using the real information.

Save the config file and quit vim with `:wq`. Then, we should create a keypair named mykey and saved it to ~/.ssh/mykey.rsa.

    starcluster createkey mykey -o ~/.ssh/mykey.rsa

This will create a [third-party keypair](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) used for connection to EC2. If the keypair (mykey) already exits, we can remove (`starcluster removekey`) and create it again, or use another name to replace 'mykey'.

After updating the \[AWS info\] and \[key mykey\] in the config file, we have the authentication to build the cluster. But we still need to add more information to the config file to satisfy our need, for example, open the port 8787 for Rstudio, and attach an EBS volume and mount it to the master node. If you want to force the master node to be spot instance, you should add `force_spot_master = true` to the config file

After all edits, the final config file will look like this:

    # in this config file, we use the master AMI for both master and slave nodes
    [global]
    DEFAULT_TEMPLATE=smallcluster

    [aws info]
    AWS_ACCESS_KEY_ID = [your access key]
    AWS_SECRET_ACCESS_KEY = [your secret access key]
    AWS_USER_ID= [your ID]

    [key mykey]
    KEY_LOCATION=~/.ssh/mykey.rsa

    [key GMFCkey]
    KEY_LOCATION = ~/.ssh/GMFCkey.rsa

    [permission www]
    from_port = 80
    to_port = 80

    [permission ftp]
    from_port = 21
    to_port = 21

    [permission Rstudio]
    from_port = 8787
    to_port = 8787

    [vol myvol]
    volume_id = [your volume id]  # created in AWS
    mount_path = /data

    [vol current]
    volume_id = [your volume id]
    mount_path = /current

    [vol GMFC]
    volume_id = [your volume id]
    mount_path = /GMFC

    [cluster smallcluster]
    KEYNAME = mykey
    CLUSTER_SIZE = 22
    CLUSTER_SHELL = bash
    NODE_IMAGE_ID = [your AMI]  # use the master AMI directly
    NODE_INSTANCE_TYPE = c3.4xlarge  # most cost effective

    MASTER_INSTANCE_TYPE = c3.4xlarge
    MASTER_IMAGE_ID =  [your AMI]

    permissions = www, ftp, Rstudio

    availability_zone = us-east-1a # optional

    #volumes = current

    force_spot_master = true

    [cluster currentcluster]
    extends = smallcluster
    volumes = current

    [cluster futurecluster]
    extends = smallcluster
    #volumes = future
             
    [cluster GMFCcluster]
    extends = smallcluster
    volumes = GMFC

4. Set Up the Cluster
---------------------

Now, everything is ready! We can build the cluster using either on-demand instance or spot instance. The on-demand instance is charged in a constant rate, and it will never be shut down. The price for spot instance varies on time based on supply and demand, and it allows you to set the maximum price you are willing to pay for the instance. The spot instance is usually much cheaper than on-demand instance, but it will be shut down once the current price is higher than your max bidding price. In this research, the shutdown of instance will not be a big problem to us, because all simulations are parallel and the results are saved to the permanent EBS volume immediately. The finished simulation will not be affected by the shutdown, and we can simply start the cluster again to continue the rest of the simulation after termination. Thus, we are going to build the cluster using spot instance to reduce cost. We can check the spot price using the following command:

    starcluster spothistory c3.large
    # this will return the price for the c3.large instance in the availability zone
    >>> Fetching spot history for c3.large (VPC)
    >>> Current price: $0.0161
    >>> Max price: $0.6000
    >>> Average price: $0.0676

We can also check the pricing history in the AWS website directly <https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#SpotInstances>: It is more flexible to check the price in the website, because it shows the price trend figures and you can change the region and availability zones to update the pricing.

Once we know the *current price*, we can star a cluster with a bidding price slightly higher than the *current price*.

    starcluster start -b 0.03 mycluster

This command will star a cluster (named mycluster) with a bidding price of $0.03 for the slave nodes. The cluster size, instance type, and availability zone information are included in the config file. Just change the config file to build a different cluster. Keep in mind that the master node will always start with the on-demand price, and it will not be turned down during the spike period. This can avoid losing the results saved in the master node. You can use the `force_master_spot = true` to force the master node to use spot instance.

In fact, the pricing is pretty stable for some instance type. For example, the price for c3.large instance is $0.0161 in the us-east-1a zone for the whole past month. If we bid with $0.02, there is little chance that the cluster will be turned down in a short time. Even if it is turned down in the middle, we can start it again after the spike or change to another region.

If you have multiple cluster templates, you can use any of them to build the cluster using command line

    starcluster start -b 0.15 -c currentcluster current

We can add more nodes to the cluster once it is established.

    starcluster addnode mycluster -n 5

The above command will add 5 nodes to mycluster. If mycluster was created with spot instances, the new nodes will also use spot instances automatically with the same bidding price.

We can ssh to the master node using the sshmaster command:

    starcluster sshmaster mycluster

Then we can open R in the terminal directly and run the simulation; or we can open Rstudio in the browser using the master's public DNS followed by *:8787* with user *jhuang* and password *rstudio*. The latter method is preferred, because it has much better interface. However, it is strange that the `qsub` command can't be called in Rstudio, but it works in R using the first method.

Inside the master node, we can check the cluster status using `qstat` and `qhost` commands. After we finish all jobs, we can just quit the master node with `exit` command and return to the starcluster host instance, from where we can stop or terminate the cluster.

    starcluster stop mycluster
    starcluster terminate mycluster

When we stop the cluster, all slave nodes will be terminated and the master node will be stopped. When we terminate the cluster, all nodes will be terminated including the master node. But in both cases, the attached EBS volume (*myvol*) will not be terminated, and it is still available for use in the next time.

Finally, we should set up the R environment in the master node, so that the Rstudio can identify the path to SGE (package used for qsub and qstat). We should edit the Renviron.site file (/usr/lib/R/etc/Renviron.site in this case. The 'R\_Home' information is included in the R&gt;system('env') output), so that we can use 'qsub/qstat' command in Rstudio. Firstly, we should login to the master node:

    starcluster sshmaster mycluster

Check the environment information with the `env` command in the terminal, and compare it with the output of `system('env')` in Rstudio, we can find that the *SGE* and its path are not included in the Rstudio environment. add the following information to the '/usr/lib/R/etc/Renviron.site' file:

    sudo vim /usr/lib/R/etc/Renviron.site 

Put the following information to the Renviron.site file

    MANPATH=:/opt/sge6/man
    SGE_CELL=default
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/opt/sge6/bin/linux-x64
    SGE_EXECD_PORT=63232
    MANTYPE=man
    SGE_QMASTER_PORT=63231
    SGE_ROOT=/opt/sge6
    DRMAA_LIBRARY_PATH=/opt/sge6/lib/linux-x64/libdrmaa.so
    ROOTPATH=:/opt/sge6/bin/linux-x64
    SGE_CLUSTER_NAME=starcluster
    LDPATH=:/opt/sge6/lib/linux-x64
    _=/usr/bin/env

quit R in Rstudio and start a new session, then we can use the 'qsub/qstat' command now.

5. Upload Data
--------------

After we set up the cluster, we should create an EBS volume, upload data to it, and attach the volume to the cluster. This attached EBS volume will work as an external disk. It will dettach from the cluster automatically when the cluster is terminated, and the data saved in the EBS volume will not disappear. To create a volume, just go to AWS console, and click *Volumes -&gt; Create Volume*. In this research, we will firstly create a 200 GB volume (inputVol). Then we can attach the volume to any instance using *Actions -&gt; Attach Volume* in the console. If the volume is attached successfully, we can see the volume information using `df -h` command in the instance's terminal. There is another way to create and attach a volume to an instance within starcluster host.

    starcluster createvolume --name=inputVol 200 us-east-1a

This command will create a volume named inputVol with 200 GB in the us-east-1a availability zone, and attach it to the host instance created in the same zone. The host instance is a t1.micro instance, which is free for first year user (720 free hours per month. So, if you have another t1.micro instance open in the same time, you will be charged). Then you can check the volume host with the listclusters function:

    starcluster listclusters volumecreator

    Launch time: 2014-10-08 03:39:29
    Uptime: 0 days, 00:05:03
    VPC: vpc-55c17330
    Subnet: subnet-3dacbe15
    Zone: us-east-1a
    Keypair: mykey
    EBS volumes:
        vol-0b526112 on volhost-us-east-1a:/dev/sdz (status: attached)
    Cluster nodes:
        volhost-us-east-1a running i-6b106985 ec2-54-86-92-150.compute-1.amazonaws.com
    Total nodes: 1

You can see that the there is no master node included in the cluster. There is only one node instance included in the cluster, which the inputVol is attached to.

To login to the volume host node, just use the sshnode function: (There seems to be an error ssh to node, the starclusterHost should be reinstalled)

    starcluster sshnode volumecreator volhost-us-east-1a

After the creation of the volume, we can terminate the volume host, so that the volume can be attached to other cluster or instance.

In this research, we will firstly attach the inputVol to starclusterHost for data uploading purpose. To attach an EBS volume, just go to AWS console, right click on the volume name, then choose attach. After attaching the inputVol, just follow this link <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html> to mount the volume to the instance. We can firstly check whether the inputVol is successfully attached with the `lsblk` command:

    lsblk
    NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0    8G  0 disk 
    xvda1   202:1    0    8G  0 part /
    xvdf    202:80   0  200G  0 disk 

Then we should check whether the file system already exists on the volume:

    sudo file -s /dev/xvdf
    /dev/xvdf: Linux rev 1.0 ext3 filesystem data, UUID=365a619d-f706-4946-8bc0-cfc6b791514b (large files)

It shows the file system already exits (/dev/xvdf is the device name). So we can go to the next step to mount the volume to a mounting point (location)

    sudo mkdir -p /data/jhuang  # create the directory recursively
    sudo mount /dev/xvdf /data/jhuang

if the file system doesn't exit, we should create firstly before going to the mounting step:

    sudo mkfs -t ext4 /dev/xvdf

or use the `-m` option to reduce the reserved block percentage (the default percentage is 5%)

    sudo mkfs.ext4 -m 1 /dev/xvdg

Now, change the permission of the /data/jhuang directory, so that we can upload data to it from within Filezilla

    sudo chmod 777 /data/jhuang

or change permission for directory and the files within it:

    sudo find /current -type d -exec chmod 777 {} \;
    sudo find /current -type f -exec chmod 777 {} \;

OK, it is done. Let's go to Filezilla and upload the data to the volume. If you upload *.rar files, you will need to use the *unrar\* package to unzip the files. Just follow this link to install the package and unrar files: <http://www.cyberciti.biz/faq/open-rar-file-or-extract-rar-files-under-linux-or-unix/>

    unrar e file.rar

After uploading and processing the data, just detach the volume from starclusterHost, so that it can be attach to other clusters. Just follow this link: <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-detaching-volume.html>

    sudo umount -d /dev/xvdf

Then exit the terminal with `exit`, and go to aws console to detach the inputVol from starclusterHost

6. Errors and FAQs
------------------

Sometimes the security group of the cluster can't be terminated. That is most probably because that the security group is used in the Network Interfaces. Checked those security groups carefully one by one, make sure no Network interface ID is using the security groups you are planning to terminate. if the problem can't be solved manually, use this small script to detect the dependency! <https://github.com/mingbowan/sgdeps> after installing the *boto*, just use the following commands to create the credential and sgdeps.py files, and put the corresponding information inside them

    sudo vim ~/.boto
    sudo vim sgdeps.py

Then run the code with the following commands:

    python sgdeps.py --region us-east-1 sg-*

7. Version Control on Git
-------------------------

After setting up the project, you may want to committee the code changes for version control purpose. The first time when you submit a change, it will ask you to enter the email and user name information. Just run the following codes inside the shell (from the Rstudio interface, go to `Tools`, then `Shell`)

    git config --global user.email jianhua.huangsdu@gmail.com
    git config --global user.name jianhuaHuang
