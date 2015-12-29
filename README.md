# R Shiny EC2 Bootstrap
The goal of this guide is to quickly bootstrap R [Shiny][] on an Amazon AWS [EC2][] instance.  We will also walk 
through a recommended workflow using Github for rapid development on a local machine and the EC2 server instance.

![image: workflow][]
    
This is an *opiniated* guide created for the following software versions:

```
Ubuntu: 14.04 LTS
R: 3.1.2
shiny-server: 1.3.0.403
```

*For other software versions, please make sure to check the version numbers used in the installations (e.g. see 
[Install Shiny Server](#install-shiny-server) section on installing different versions of shiny-server).*

----------

## Contents
* [Launch EC2](#launch-ec2)
* [Register Key Pair](#register-key-pair)
* [Create and enable swap file](#create-and-enable-swap-file)
* [Install R](#install-r)
* [Install R Packages](#install-r-packages)
* [Install Shiny Server](#install-shiny-server)
* [Install Git](#install-git)
* [Workflow Overview](#workflow-overview)
* [Problematic Packages](#problematic-packages)
* [Examples](#examples)
* [Resources](#resources)

----------

## Launch EC2
-   Create an [AWS][] account and login to the [console][].

-   Click on `Compute > EC2` (near the top left of the screen) and click the `Launch Instance` button in the `Create 
    Instance` section of the page.

-   We are now in the AMI (Amazon Machine Image) interface.  Select `Ubuntu Server 14.04 LTS`.

    ![image: ubuntu14.04][]

-   We will choose the default `t2.micro` instance.  Accept all the default settings for the next few steps:
    -   Click `Next: Configure Instance Details`
    -   Click `Next: Add Storage`
    -   Click `Next: Tag Instance`
    -   Click `Next Configure Security Group`

-   When we arrive to `Configure Security Group`, we will need to add a few more security groups to open our EC2 
    instance to the world when we later build applications with Shiny.

-   Add the following rules using the `Add Rule` buttons.  This registers and opens the ports `3838` and `80` 
    that Shiny will use later.  By default, Shiny will listen on the `3838` port, but keeping the default HTTP 
    listening port `80` open is always a useful option (In the [Install Shiny Server](#install-shiny-server) section,
     we go through changing the `shiny-server` configurations to listen on the default `80` port, which removes the 
     need to specify a port number in your URL).

    ![image: configure security groups][]
    
-   We are ready to launch our EC2 instance now.

-   Click on the `Review and Launch` button, and `Launch` your instance!

    ![image: ec2 launch][]

<sub>([back to contents](#contents))</sub>

----------

## Register Key Pair
-   With our EC2 instance set up, we need to register the key provided to us from Amazon with our local machine. 

-   Amazon will prompty you to create a new key pair.  For this guide, we shall name the keypair `shinybootstrap`.

    ![image: create key pair][]
    
-   Download the `shinybootstrap.pem` key pair.  We recommend that you place your `shinybootstrap.pem` key pair in a 
    safe folder.  [Ekuns](https://github.com/ekuns) has provided a great
    [reuseable and general tip][general ssh] on setting up and
    managing `ssh` connections in general.
    
    - Copy the `shinybootstrap.pem` key into the `~/.ssh` folder
    - Create and edit the file using a text editor in `~/.ssh/config` (config is the filename)
    
        ```
        Host *
            ServerAliveInterval 240
    
        Host aws-shiny
            HostName public_dns_name
            User ubuntu
            IdentityFile ~/.ssh/aws-shiny.pem
            ForwardX11 yes
            ForwardX11Trusted yes
        ```
    
    - where `aws-shiny` is the alias hostname you are providing and `public_dns_name` is the name of your EC2 instance
      e.g. `ec2-54-183-2-10.us-west-1.compute.amazonaws.com` or the public IP itself `54.183.2.10`
      
      ![image: dns name][]
    
    - Restart your terminal and you can now access your ubuntu EC2 server using:
    
        ```
        ssh aws-shiny
        ```
    
    -   **NOTE:** The public DNS name and public IP may not be static/permanent, e.g. their values can change when 
        `shiny-server` restarts.  For a full permanent solution to creating a `bash_alias`, you would need to create 
        an [Elastic IP][] (but we will defer that for the readers to the Amazon guide if readers wish to research the
        details).
    

<sub>([back to contents](#contents))</sub>

----------

## Create and enable swap file

-   This step is necessary when running on a T2.mirco EC2 instance. You won't be able to install certain R packages (e.g. `dplyr`, `tidyr`) without the additional allocated memory from the swap 

-  First we need to create and enable the swap file:

    ```bash
    sudo fallocate -l 1G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    ```
    
-   Next, we need to make sure that the swap file is enabled even after rebooting the system

-   Edit the `fstab``file with root privileges in your text editor:

    ```bash
    sudo nano /etc/fstab
    ```

-   At the bottom of the file, you need to add a line that will tell the operating system to automatically use the file you created:

    ```
    /swapfile   none    swap    sw    0   0
    ```

-   Save and close the file when you are finished.

      
<sub>([back to contents](#contents))</sub>   
    
----------



## Install R
-   Phew! The tough parts are over, we can now move onto installing R in our EC2 instance!

-   First, create a superuser root password by entering a password in the EC2 command line.
    
    ```bash
    sudo password root
    su
    ```
    
-   Update your barebones EC2 virtual machine and install R:
    
    ```bash
    sudo apt-get update
    sudo apt-get install r-base-dev
    sudo apt-get install libcurl4-openssl-dev
    ```
    
-   R is now successfuly installed!

<sub>([back to contents](#contents))</sub>

----------

## Install R Packages
-   Base R is argubly boring, so let's get started with installing a few important packages, `shiny` being an 
    obviously shiny option.  (*while installing these packages, make sure you are in `su` mode and I highly recommend
    installing them from the `0-Cloud` CRAN repository*)

-   From your EC2 command line:
    
    ```bash
    R       # enter the R console
    
    install.packages("shiny")
    install.packages("ggplot2")
    q()         # quit R
    ```
    
-   A few other packages are highly recommended but you might encounter issues installing certain packages due to 
    memory limitations available on the EC2 instance or missing libraries in our EC2 instance.  For more details, 
    please refer to the [Problematic Packages](#problematic-packages) section for a brief outline.

-   Other than that, you can come back anytime in the future to install R-packages by logging in with `su` and 
    installing R packages via the command line.

<sub>([back to contents](#contents))</sub>

----------

## Install Shiny Server
-   `shiny` requires a server to run.  The next step is to setup `shiny-server` in our EC2 instance.

-   Run the following commands in EC2:
    
    ```bash
    sudo apt-get install gdebi-core
    wget http://download3.rstudio.org/ubuntu-12.04/x86_64/shiny-server-1.3.0.403-amd64.deb
    sudo gdebi shiny-server-1.3.0.403-amd64.deb
    ```

-   **NOTE:** You can install a different version of `shiny-server` in the above command prompt by adjusting the 
    `shiny-server-version_number` in the `wget` and `gdebi` commands above.  You can check for the latest versions of
     [RStudio shiny-server](https://www.rstudio.com/products/shiny/download-server/) on the official site.

-   **OPTIONAL:** *Change the listening port from `3838` to default `80`*
    -   `shiny-server` by default runs applications on the URL and port: `http://domain:3838/`.  We can take the time
        to make an easy change to the `shiny-server` `config` file to listen to the default port at `80`.  This will 
        allow our future apps to be accessed without specifying a port number by simply going to `http://domain/`

    -   Execute the following commands in your EC2 instance:
    
        ```bash
        cd /etc/shiny-server/ 
        sudo nano shiny-server.conf
        ```
    
    -   At this point, you can use a text editor (unfortunately we can only use CLI and non-GUI text editors) to 
        change the line `listen 3838;` to `listen 80;`
        
    -   Makes sure to save the changes and you can now access your shiny instances without specifying the `3838` port
        number!
        
- And we're done!

<sub>([back to contents](#contents))</sub>

----------

## Install Git
-   `git` is probably one of the greatest creations and tools that you can use in your life, so it's good to include 
    it in our EC2 instance.

-   Installing `git` is as simple as typing:
   
    ```sh
    sudo apt-get install git
    ```
    
-   `git` is very critical in our Shiny EC2 workflow.  If you are not familiar with using `git`, here are some great 
    resources to get on speed:
    -   [Code School][]
    -   [Official Git][]

<sub>([back to contents](#contents))</sub>

----------

## Workflow Overview
-   A big part of the EC2 R Shiny workflow is to develop with RStudio on your local machines and use Github as a 
    repository station for transferring data between your local machine and EC2 instance.  Make sure that you are
    familiar with the basic `git` commands: `pull`, `push`, `clone`.  It's going to be a big part of the development 
    cycle.

-   The following diagram outlines the overall workflow of the EC2 R Shiny workflow:
    
    ![image: workflow][]

-   Workflow Summary:
    1.  Develop on your local machine with RStudio.
    2.  Whenever you are ready to show your work to the world, you would `git push` your changes to your Github 
        repository.
    3.  Initiate your EC2 instance with the same codebase on Github by using `git clone` into `/srv/shiny-server/` 
        (this is the folder where `shiny-server` serves your application).
    4.  If there are changes to be made to your app, repeat steps 1 and 2 to `git push` updates to Github from your 
        local machine, and then perform a `git pull` in your EC2 instance to update your application in EC2.

<sub>([back to contents](#contents))</sub>

----------

## Problematic Packages
-   Since our EC2 instance is completely barebones, installing various packages might be problematic due to a lack of
    libraries that EC2 comes with.

-   My recommendation is to do searches on [Stackoverflow][] and [Google][] using keywords such as `EC2`, 
    `package-name`, `installation`, `R`.

-   You could run into issues with missing Python libraries and version requirements for R and various packages, but 
    most of them can be solved by searching on Stackoverflow and Google.


-   Stackoverflow + Goole are most definitely the best teachers and guides out there for resolving problematic 
    packages.  My job here is to just inform on some of the issues you may encounter :)

<sub>([back to contents](#contents))</sub>

----------

## Examples
Here are some project examples of Shiny EC2 instances that I have built and their respective Github codebase.

-   State Crime Rates
    <sub>([EC2][example1-ec2] | [Github][example1-github])</sub>
-   Labor Force Statistics 
    <sub>([EC2][example2-ec2] | [Github][example2-github])</sub>
-   Box Office Mojo 
    <sub>([EC2][example3-ec2] | [Github][example3-github])</sub>
-   Power to Choose 
    <sub>([EC2][example4-ec2] | [Github][example4-github])</sub>

<sub>([back to contents](#contents))</sub>

----------

## Resources
This guide would not be possible without the works and guides provided by previous people.  I'm listing some 
additional resources that are helpful for the general audience.  Please contact me if there are any 
issues/corrections with this guide.  Thank you!

-   Shiny AWS EC2 Guides:
    -   [Tyler Hunt's guide to set up Shiny AWS EC2][tyler hunt]
    -   [Custom AWS inbound rules][aws inbound rules]
    
-   Digital Ocean Guides
    -   [How To Add Swap on Ubuntu 14.04][do swap]

-   Other resources:
    -   [Official RStudio shiny-server guide][rstudio shiny-server guide]
    -   [General SSH config][general ssh]
    -   [Creating ElasticIP for maintaining "permanent" IP addresses][elastic ip]
    -   [AWS Route 53 to rename EC2 instances to domain address][aws route 53]
    -   [Stackoverflow][]
    -   [dplyr][]

<sub>([back to contents](#contents))</sub>

----------

<!-- links -->
[shiny]: http://shiny.rstudio.com/
[ec2]: http://aws.amazon.com/ec2/
[aws]: https://aws.amazon.com/
[console]: https://console.aws.amazon.com
[elastic ip]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html
[rttudio shiny-server]: http://www.rstudio.com/products/shiny/download-server/
[code school]: https://www.codeschool.com/courses/try-git
[official git]: http://git-scm.com/
[stackoverflow]: http://stackoverflow.com/
[google]: http://google.com/

<!-- resource section links -->
[tyler hunt]: http://tylerhunt.co/2014/03/amazon-web-services-rstudio/
[aws inbound rules]: http://www.r-bloggers.com/deploying-shiny-server-on-amazon-some-troubleshoots-and-solutions/
[do swap]: https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04
[rstudio shiny-server guide]: http://rstudio.github.io/shiny-server/latest/
[aws route 53]: http://aws.amazon.com/route53/
[dplyr]: http://cran.rstudio.com/web/packages/dplyr/vignettes/introduction.html
[General SSH]: https://github.com/chrisrzhou/RShiny-EC2Bootstrap/issues/2

<!-- example section links -->
[example1-ec2]: http://shiny.vis.datanaut.io/StateCrimeRates/
[example2-ec2]: http://shiny.vis.datanaut.io/LaborForceStatistics/
[example3-ec2]: http://shiny.vis.datanaut.io/BoxOfficeMojo/
[example4-ec2]: http://shiny.vis.datanaut.io/PowerToChoose/
[example1-github]: https://github.com/chrisrzhou/RShiny-StateCrimeRates
[example2-github]: https://github.com/chrisrzhou/RShiny-LaborForceStatistics
[example3-github]: https://github.com/chrisrzhou/RShiny-BoxOfficeMojo
[example4-github]: https://github.com/chrisrzhou/RShiny-PowerToChoose

<!-- s3 image links -->
[image: workflow]: https://s3-us-west-1.amazonaws.com/chrisrzhou/github/RShiny-EC2Bootstrap/aws_ec2_workflow.png
[image: ubuntu14.04]: https://s3-us-west-1.amazonaws.com/chrisrzhou/github/RShiny-EC2Bootstrap/aws_ec2_ubuntu14.04.png
[image: configure security groups]: https://s3-us-west-1.amazonaws.com/chrisrzhou/github/RShiny-EC2Bootstrap/aws_ec2_configure_security_groups_complete.png
[image: ec2 launch]: https://s3-us-west-1.amazonaws.com/chrisrzhou/github/RShiny-EC2Bootstrap/aws_ec2_launch.png
[image: create key pair]: https://s3-us-west-1.amazonaws.com/chrisrzhou/github/RShiny-EC2Bootstrap/aws_ec2_create_keypair.png
[image: dns name]: https://s3-us-west-1.amazonaws.com/chrisrzhou/github/RShiny-EC2Bootstrap/aws_ec2_dns_name.png
