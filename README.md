# RStudio Server on Google Compute Engine

This is a how-to guide for setting up a server or virtual machine (VM) with [Google Compute Engine](https://cloud.google.com/compute/). In addition, I'll also show you how to install RStudio Server on your VM, so that you can perform your analysis in almost exactly the same user environment as you're used to, but now using the full power of cloud-based computation. Trust me, it will be awesome.

## Prerequisites

1. Sign up for a [60-day free trial](https://console.cloud.google.com/freetrial) with the the Google Cloud Platform. This requires an existing Google/Gmail acount, although UCSB employees can just use their university email address, since that is run through Gmail. You'll also need to create a project that will be associated with billing. (Purely for ceremony at present because we're using the free trial period.)
2. Download and follow the installation instructions for the Google Cloud SDK command line utility, `gcloud` [here](https://cloud.google.com/sdk/).

## Introduction

First things first: What is a [virtual machine (VM)](https://en.wikipedia.org/wiki/Virtual_machine) and why do I need one anyway? In the simplest sense, a VM is just an emulation of a computer running inside another (bigger) computer. It can potentially perform all or more of the operations that your physical laptop/desktop does, and it might have many of the same properties (from operating system to internal architecture.) The key advantage of a VM from our perspective here is that very powerful machines can be "spun up" in the cloud almost effortlessly and then deployed to tackle jobs that are beyond the capabilities of your local computer. Got a big dataset that requires too much memory to analyse on your old laptop? Load it into a high-powered VM. Got some code that takes an age to run? Fire up a VM and let it chug away without consuming any local resources. Or, better yet, write the code in parallel and then spin up a VM with lots of cores (CPUs) to get the analysis done in a fraction of the time. All you need is a working internet connection and a web browser.

Now, with that bit of background in mind, Google Compute Engine is part of the [Google Cloud Platform](https://cloud.google.com/) and delivers high-performance, rapidly scalable VMs. A new VM can be deployed or shut down within seconds, while exisiting VMs can easily be ramped up or down (cores added, RAM added, etc.) depending on a project's needs. In my experience, Google Compute Engine is at least as good as Amazon AWS -- say nothing of the [other really cool products](https://cloud.google.com/products/) within the Cloud Platform suite -- and most individual users would be really hard-pressed to spent more than a couple of dollars a month using it. (If that.) This is especially true for the researcher who just needs to crunch some large dataset or run some simulations, and can easily switch the machine off when it's not being used.

Two final housekeeping notes, before continuing.

First, it's possible to complete nearly all of the steps in this guide via the [Compute Engine browser console](https://console.cloud.google.com/home/dashboard). However, we'll stick with the `gcloud` command line utility (which you should have [installed](https://cloud.google.com/sdk/) already), because that will make it easier to [document our steps](http://remi-daigle.github.io/shell/) and will also save us some headaches further down the road. For example, when it comes to transferring files between your local computer and a Compute Engine instance.

Second, almost all VMs run on some variant of Linux. Since we'll only be connecting to our VM instance via the terminal, this only matters insofar as some of the commands might invoke slightly different syntax to what you'd normally use on a Mac or Windows PC. If you're brand new to Linux, then I'd recommend taking a quick look at [this website](https://linuxjourney.com/). It provides a great step-by-step overview of some of the key concepts and commands. One thing that I'll briefly mention here is that Ubuntu -- the Linux distribution that we'll be using below -- uses the `apt` package-management system. (Much like Mac OS uses Homebrew.) So when you see commands like `apt-get install PACKAGENAME`, that's just a convenient way to install and manage packages.

Okay, introduction out of the way. Let's get up a running.

## Setup and log-in

You'll need to choose an operating system for your VM, as well as the server zone (region). To see the available options, first open up the terminal ([Windows](http://www.digitalcitizen.life/7-ways-launch-command-prompt-windows-7-windows-8), [Mac](https://www.techwalla.com/articles/how-to-open-terminal-on-a-macbook), [Linux](http://www.wikihow.com/Open-a-Terminal-Window-in-Ubuntu)). Then enter
```
~$ sudo gcloud compute images list
~$ sudo gcloud compute zones list
```

We'll go with Ubuntu 16.04 and set our zone to the U.S. west coast. You can also choose a bunch of other options by using the appropriate flags -- see [here](https://cloud.google.com/sdk/gcloud/reference/compute/instances/create). I'm going to call my VM instance "rstudio" but you can obviously call it whatever you like. I'm also going to specify the type of machine that I want. In this case, I'll go with the `n1-standard-8` option (8 CPUs with 30GB RAM), but you can choose from a [range](https://cloud.google.com/compute/pricing) of machine/memory/pricing options. (A cool feature of Compute Engine is that it is very easy to change the specs of your VM and Google will even suggest cheaper alternatives if it thinks that you aren't using your resource capabilities efficiently over time.) In the terminal window, type:
```
~$ sudo gcloud compute instances create rstudio --image-family ubuntu-1604-lts --image-project ubuntu-os-cloud  --machine-type n1-standard-8 --zone us-west1-a
```

This should generate something like:
```
Created [https://www.googleapis.com/compute/v1/projects/YOUR-PROJECT/zones/us-west1-a/instances/rstudio].
NAME      ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
rstudio  us-west1-a  n1-standard-8               10.138.0.2   104.198.105.102  RUNNING
````

Write down the External IP address, as we'll need it for running RStudio server later. (It is also possible to assign a static, external IP address to your VM instance that is more memorable. See [here](https://cloud.google.com/compute/docs/configure-instance-ip-addresses#assign_new_instance). ) On a similar note, RStudio Server will run on port 8787 of the External IP, which we need to enable via the Compute Engine firewall.
```
~$ sudo gcloud compute firewall-rules create allow-rstudio --allow=tcp:8787
```
Congrats: Set-up for your Compute Engine VM instance is complete! Easy, wasn't it?

Let's start it up and then log in via SSH. This is a simple matter of providing your VM's name and zone (if you forget to specify the zone, you'll be prompted):

```
~$ sudo gcloud compute instances start rstudio --zone us-west1-a
~$ sudo gcloud compute ssh rstudio --zone us-west1-a
```

Upon logging in for the first time, you will be prompted to generate an SSH key passphrase. Needless to say, you should make a note of this for future long-ins. You should now be connected to your VM via terminal. Next, we'll install *R* before moving on to RStudio Server.

### Install *R* on your VM

You can find the full set of instructions and recommendations for installing *R* on Ubuntu [here](https://cran.r-project.org/bin/linux/ubuntu/README). Or you can just follow my choices below, which covers everything that you should need.
```
~# sh -c 'echo "deb https://cloud.r-project.org/bin/linux/ubuntu xenial/" >> /etc/apt/sources.list'
~# apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E084DAB9
~# apt-get update
~# apt-get install r-base r-base-dev
```
In addition to the above, a number of important *R* packages (e.g. `curl`, which in turn is a dependency for many other packages) require external Linux libraries that must first be installed separately on your VM. In Ubuntu (or other Debian-based distros), run the below commands in your VM's terminal:

1) For the "tidyverse" suite of packages (i.e. `install.packages("tidyverse")`:
```
~# apt-get install libcurl4-openssl-dev libssl-dev libxml2-dev
```
2) For the main spatial libraries (sp, rgeos, etc.):
```
~# apt-get install libgeos-dev libproj-dev libgdal-dev
```

*R* is now ready to go your VM directly from terminal:
```
~# R
```
However, we obviously want to use RStudio (Server). So that's what we'll install and configure next, making sure that we can run RStudio Server on our VM via a web browser like Chrome or Firefox from our local computer. (Hit `q()` and then `n` to exit the terminal version of *R* if you opened it above.)

## Install and configure RStudio Server

### Download RStudio Server on your VM

You should check what the latest available version of Rstudio Server is [here](https://www.rstudio.com/products/rstudio/download-server/), but as of today (11 Jan 2016) the following is what you need:
```
~# apt-get install gdebi-core
~# wget https://download2.rstudio.org/rstudio-server-1.0.136-amd64.deb
~# gdebi rstudio-server-1.0.136-amd64.deb
```

### Add a user

Now that you're logged into your VM, you might notice that you haven't actually signed in as a specific user. In fact, you're signed into the "rstudio" VM as root. (Fun fact: You can tell because the command line has a hashtag instead of a dollar sign.) This doesn't matter for most applications, but RStudio Server specifically requires a username/password combination. So we first need to create a new user before continuing. For example, to create a new user called "elvis" enter the follow command in terminal.
```
~# adduser elvis
```
You will then be prompted to specify a password for this user (and confirm various bits of biographical information which you can largely ignore).

### Navigate to your RStudio Server instance in your browser

You are now ready to open up RStudio Server by navigating to the default 8787 port of your VM's External IP address. (You remember writing this down earlier, right?) If you forgot to write the IP address down, don't worry: You can find it by logging into your Google Cloud console and looking at your [VM instances](https://console.cloud.google.com/compute/instances), or by opening up a new terminal window (not the one currently running your VM) and typing
```
sudo gcloud compute instances describe rstudio  --zone us-west1-a
```
Either way, once you have the address, open up your preferred web browser and navigate to:
```
http://<external-ip-address>:8787
```
You will be presented with the following web page:

![](./pics/rstudio-server-login.png)

Log in using the unix username/password that you created earlier and you're set. (Tip: Hit F11 to go full screen in your browser. The server version of RStudio is then almost indistinguishable from the desktop version.)

### Reading and writing files to and from RStudio Server

There's a slight wrinkle in the above set-up. Namely, RStudio Server is only going to be able to look for files in our user's home directory (e.g. `/home/elvis`.) The reason has to do with user permissions; since Elvis is not a "super user", RStudio server doesn't know that (s)he is allowed to access other directories in our VM. Thankfully, there's a fairly easy workaround, involving standard Linux commands for adding [user and group](https://linuxjourney.com/lesson/users-and-groups) [privileges](https://linuxjourney.com/lesson/file-permissions). I won't explain these in depth here, but let's just say that we want to keep all of our analysis in a new directory called "Papers". First create this directory:
```
$ sudo mkdir Papers
```
Next, create a group (I'll call it "papersgrp"), whose members should all have full read, write and execute access to files within the Papers directory. Then add both the default user (which should be root) and elvis to this group:
```
$ sudo groupadd papersgrp
$ sudo gpasswd -a <defaultuser>
$ sudo gpasswd -a elvis
```
Next, set our default user and the other papersgrp members as owners of this directory (`chown -R`) as well as all of its children directories. Grant them all read, write and execute access (`chmod -R 770`):
```
$ sudo chown -R <defaultuser>:papersgrp Papers
$ sudo chmod -R 770 Papers
```

The next two commands are optional, but advised. Since Elvis (or whatever your username is) should only be working with files in the Papers directory, you can change his/her primary group to papersgrp, so that all the files (s)he creates are automatically assigned to that group:
```
$ sudo usermod -Papers papersgrp elvis
```
Finally, you can add a symbolic link to the Paper directory in Elvis’s home directory, so that it is immediately visible when you log into RStudio Server. (Make sure that you switch to this user before running this command):
```
$ sudo usermod -Papers papersgrp elvis
$ ln -s /home/<defaultuser>/Papers /home/elvis/Papers
```

## Transferring and syncing files between your VM and your local PC

You have two main options.

### 1. Manually transfer files and folders using the command line or SCP

Manually transferring files or folders across systems is fairly easily done using the command line:

```
sudo gcloud compute copy-files rstudio:/home/elvis/Papers/MyAwesomePaper/amazingresults.csv ~/local-directory/amazingresults-copy.csv --zone us-west1-a
```
It's also possible to transfer files using your regular desktop file browser thanks to SCP. (On Linus and Mac OSX at least. Windows users first need to install a program call WinSCP.) See [here](https://cloud.google.com/compute/docs/instances/transfer-files).

### 2. Sync with Git(Hub), Box, Dropbox, or Google Drive

Ubuntu, like all Linux distros, comes with Git preinstalled. You should thus be able to sync your results across systems using Git(Hub) in the [usual fashion](http://happygitwithr.com/). I tend to use the command line for all my Git operations -- committing, pulling, pushing, etc. -- and I also had some teething problems with Rstudio Server's Git UI when I first tried it on a VM. However, I believe that these issues have been mostly resolved so let me know if that works for you.

Similarly, while I haven't tried it myself, you should also be able to install [Box](http://xmodulo.com/how-to-mount-box-com-cloud-storage-on-linux.html), [Dropbox](https://www.linuxbabe.com/cloud-storage/install-dropbox-ubuntu-16-04) or [Google Drive](http://www.techrepublic.com/article/how-to-mount-your-google-drive-on-linux-with-google-drive-ocamlfuse/) on your VM and sync across systems that way. (Fair warning: Remember that your VM lives on a server and doesn't have the usual graphical interface -- including installation utilities -- of a normal desktop. You'll thus need to follow command line installation instructions for these programs. Make sure you scroll down to the relevant sections of the links that I have provided above.) If you go this way, then I'd also suggest that you follow the instructions for linking to the "Papers" folder above, except that you now point towards the user's relevant Box/Dropbox/GDrive folder.

Last, but not least, Google themselves encourage data synchronisation on Compute Engine VMs using another product within their Cloud Platform, i.e. Google Storage. This is especially for really big data files and folders, but beyond the scope of this tutorial. (If you're interested in learning more, see [here](https://cloud.google.com/solutions/filers-on-compute-engine) and [here](https://cloud.google.com/compute/docs/disks/gcs-buckets).)

## Stopping and (re)starting your VM instance
Stopping and (re)starting your VM instance is easy, so you don't have to worry about getting billed for times when you aren't using it.
```
~$ sudo gcloud compute instances stop rstudio
~$ sudo gcloud compute instances start rstudio
```

## Other tips
Remember to keep your VM system up to date.
```
~# gcloud components update
~# apt-get upgrade
```

## Post-installation summary and additional resources

Assuming that you have gone through the initial set-up, here's a quick **tl;dr** summary for accessing an existing VM (via RStudio Server) in the future:

1) Start-up a VM instance.
```
~$ sudo gcloud compute instances start YOUR-VM-INSTANCE-NAME --zone us-west1-a
```
2) Log-in via SSH.
```
~$ sudo gcloud compute ssh YOUR-VM-INSTANCE-NAME --zone us-west1-a
```
3) Take note of the External IP address above, or type (in a new terminal window):
```
~$ sudo gcloud compute instances describe YOUR-VM-INSTANCE-NAME  --zone us-west1-a
```
4) Open up a web browser and navigate to the following address (enter your username/password as needed):
```
http://<external-ip-address>:8787
```
5) Stop your VM:
```
~$ sudo gcloud compute instances stop YOUR-VM-INSTANCE-NAME --zone us-west1-a
```
And, remember, if you really want to avoid the command line, then you can always go through the [Compute Engine browser console](https://console.cloud.google.com/home/dashboard).

Lastly, if you ever get stuck, then just consult the relevant documentation. There's tonnes of useful advice and extra tips for getting the most out of your VM setup.
- Google Compute Engine documentation ([link](https://cloud.google.com/compute/docs/))
- RStudio Server documentation ([link](https://support.rstudio.com/hc/en-us/articles/234653607-Getting-Started-with-RStudio-Server))
