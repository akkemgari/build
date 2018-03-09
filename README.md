# README #

### What is this repository for? ###

* Building xx_core images
* Uploading other FIBR images to EC2
* General management of AWS build environment


### Linux Build Process Overview ###

* Packer (AWS and Vagrant)
    * Builder phase: Creates a VirtualBox VM and installs Linux via Kickstart
    * Provisioner phase: Adds some FIBR yum repos, installs software, applies a basic security baseline, patches, and disables non-required services. Also does further configuration based on target environment (e.g. CloudFormation for AWS, setting up the user and key for Vagrant). Outputs an ova file for AWS.
    * Post-processor phase: Outputs a box file for Vagrant, and creates a manifest file listing all of the ova and box files produced.
* AWS CLI (AWS only)
    * Uploads each ova file to S3, imports it into an EC2 AMI, spins up an instance from that AMI, sets the "Delete on termination" property on the root volume of the instance, creates a new AMI from it, and deletes the instance and original AMI.
    * Copies the AMI to all other regions.
    * Adds all FIBR AWS accounts to the launch permissions for each AMI, so everyone can run them.
    * Will also set standard tags on each AMI, once we're granted access to do so.


### Build Process Details ###

Note: The build process is being modified and extended much more frequently than this readme is revised, so if you want to understand all the minutiae of the build process, reading the comments in the various config files and scripts is recommended.

Packer is configured by way of some environment variables and a json file with the specific workflow for each build. Currently our packer workflow is virtualbox-iso builder --> remote shell provisioners --> vagrant post-processor (Vagrant only). With 4 OSes (CentOS/RHEL 6/7) and 2 target environments (AWS/Vagrant), this means there are 8 builds total. (Plus a couple test workflows, while the process is further refined.)

The Kickstart files used by the builder are derived from the ones we use for on-premise builds, but simplified and with modifications necessary for the conversion to AMI. Aside from disk partitioning, the main difference is that RHEL/CentOS 7 require more services to be running than what we normally use, or they will stall indefinitely in the "booting" step of the import. (Or for many days, anyway; I think the longest I've left one running is 5 days, and it never completed or failed in that time.) This step should only take a few minutes, so if it's still "booting" after a half-hour, you can safely conclude that it is broken and cancel the import. The additional services required to be enabled are: kdump, libstoragemgmt, mdmonitor, sysstat, tuned, and network.

There is a common shell provisioning script which runs for all builds, that turns off the default requirement for a tty for sudo (causes problems for chef), adds a few basic repos from the prod tag, runs yum update, installs a few extra packages, and disables non-required services. For AMI builds, prerequisites for CloudFormation are installed, CloudFormation is built from the latest sources, the AWS CLI is installed, an ec2-user is created and added to sudoers, cloud-init is installed, and an xx-build.yaml file is created with some basic build details. For Vagrant builds, a vagrant user is created and added to sudoers, the standard vagrant key is added (which vagrant uses and then replaces when it brings up a box), and an xx-build.yaml file is created with some basic build details.

The AWS CLI piece is a list of commands that are run to upload the ova file to S3, import it from S3 to an EC2 AMI, copy the AMI to any other regions, set launch permissions for that AMI to all NIBR accounts in all regions, and set standard tags everywhere. This is about 3/4 automated currently, with some bugs remaining to be worked out with the tags.


### How do I get set up? ###
To run builds, etc on the current build server:
1. Log in to na-lx41002
2. If you're going to be starting a build or any other potentially lengthy process, run screen first
3. su to root
4. source the build environment script: `. /apps/xx_build/env.sh`

To create a new build server:
1. Check out this repo
2. Set the correct locations for the variables in main env.sh and init/env.sh
3. Run init/init.sh
Note that, while this is all code that has been used creating the current build server, making a wholly new one hasn't been done yet, so expect some bugs or hiccups if you try it.


### Okay I'm set up. How do I do X?
* Create a new full set of nx_core builds:
    * Run `build_images`

* Upload an image provided in ova format:
    * Peruse the code in env.sh, then run the commands that it would run for nx_core, substituting the provided ova file and associated metadata
    * Yes, this will be easier in the future. It's just competing with a few dozen other items for development time

* Perform other management tasks:
    * Take a look at the AWS CLI documentation: http://docs.aws.amazon.com/cli/latest/reference/
    * Poke around in env.sh for examples of how some of the commands are used
    * Try not to break anything


### Contribution guidelines ###

* If you have a request for something to be added to the nx_core build or to the general image upload process, create a ticket in the NXCR project in JIRA, and assign to snydesc1
* If you are directly modifying or adding to the process, be aware that on nrusca-sld4002 (the current build server), this repo is checked out under root to snydesc1, so you'll need to either change that or bug snydesc1 to push/pull your changes


### Who do I talk to? ###
