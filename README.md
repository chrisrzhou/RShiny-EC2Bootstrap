# R Shiny EC2 Bootstrap
The goal of this guide is to quickly bootstrap an [Amazon EC2](http://aws.amazon.com/ec2/) instance with R and [Shiny](http://shiny.rstudio.com/) set up correctly.

----------
## Contents
* [Launch EC2](#launch-ec2)
* [Register Key Pair](#register-key-pair)
* [Install R](#install-r)
* [Install R Packages](#install-r-packages)
* [Install Shiny Server](#install-shiny-server)
* [Install Git](#install-git)
* [Workflow Overview](#workflow-overview)
* [Problematic Packages](#problematic-packages)
* [Resources](#resources)

----------
## Launch EC2
- Create an AWS account and login to the [console](https://console.aws.amazon.com)
- Click on the `EC2` option and click the `Launch Instance` button.
- We are now in the AMI (Amazon Machine Image) interface.  Select `Ubuntu Server 14.04 LTS`

    ![alt Amazon Machine Image](./images/aws_ec2_ubuntu4.04.png)


[(back to contents)](#contents)

----------
## Register Key Pair
- With our EC2 instance set up, we need to register the `.pem` key provided to us from Amazon with our local machine. (*Note that only one EC2 key can be registered to one machine.*)
- We recommend that you place your `.pem` file in a safe folder, but for purposes of this guide, we will assume that you are placing the file in the folder `~/sshKeys`.
    ```
    # create sshKeys folder in home directory
    mkdir ~/sshKeys
    
    # move the .pem file to the sshKeys folder
    mv shinybootstrap.pem ~/sshKeys
    
    # grant sudo access to .pem file
    chmod 400 ~/sshKeys/shinybootstrap.pem
    ```
- **Note**: If you ever run into a `Permission denied (publickey)` error message, this is because you are not able to access the `.pem` file.  Make sure that you have granted sudo access to the file using `chmod 400` as outlined above.
- It is useful to create an alias to access the EC2 instance.  We shall create the alias as `awslogin` and register this in the `bash_aliases` file using the `echo "statement" >> ~/.bash_aliases` command:
    ```
    echo "alias awslogin='ssh -i ~/sshKeys/shinybootstrap.pem ubuntu@ec2-54-183-2-10.us-west-1.compute.amazonaws.com'" >> ~/.bash_aliases
    ```
- Restart your terminal to access the changes and you should connect to your EC2 instance simply by typing `awslogin` in the terminal console.

[(back to contents)](#contents)

----------
## Install R
- Phew! The tough part is over, we can now move onto installing `R`!
- First, create a superuser root password by entering a password after the following commands.
    ```
    sudo password root
    su
    ```
- Update your barebones EC2 virtual machine and install the following:
    ```
    sudo apt-get update
    sudo apt-get install r-base-dev
    sudo apt-get install libcurl4-openssl-dev
    ```
- R is now successfuly installed!

[(back to contents)](#contents)

----------
## Install R Packages
- Base R is argubly boring, so let's get started with installing a few important packages, `shiny` being an obviously shiny option.  (*while installing these packages, make sure you are in `su` mode and I highly recommend installing them from the `0-Cloud` CRAN repository*)

    ```
    R       # enter the R console
    
    install.packages("shiny")
    install.packages("ggplot2")
    q()         # quit R
    ```
- A few other packages are highly recommended but you might encounter issues installing certain packages due to memory limitations available on the EC2 instance.  I have provided some uncompiled files for some of these packages (e.g. `dplyr`) and some details on working with these packages under the section **[Problematic Packages](#problematic-packages)**.

[(back to contents)](#contents)

----------
## Install Shiny Server
- `shiny` requires a server to run.  The next step is to setup `shiny-server` in our EC2 instance.
- Run the following commands:
    ```
    sudo apt-get install gdebi-core
    wget http://download3.rstudio.org/ubuntu-12.04/x86_64/shiny-server-1.0.0.42-amd64.deb
    sudo gdebi shiny-server-1.0.0.42-amd64.deb
    ```
- And we're done!


[(back to contents)](#contents)

----------
## Install Git
- `git` is probably one of the greatest creations and tools that you can use in your life, so it's good to include it in our EC2 instance.
- Installing `git` is as simple as typing:
    ```
    sudo apt-get install git
    ```

[(back to contents)](#contents)

----------
## Workflow Overview
- A big part of the EC2 R Shiny workflow is to develop with RStudio on your local machines and use Github as a repository station for transferring data between your local machine and EC2 instance.  Make sure that you are familiar with the basic `git` commands: `pull`, `push`, `clone`.  It's going to be a big part of the development cycle.
- The following diagram outlines the overall workflow of the EC2 R Shiny workflow:



[(back to contents)](#contents)


----------
## Problematic Packages



[(back to contents)](#contents)

----------
## Resources
