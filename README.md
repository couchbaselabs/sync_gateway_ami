Scripts to create couchbase amazon ec2 AMI's.

# Prerequisites

You'll need....

* java (for the amazon ec2 API tools)
* JAVA_HOME environment variable properly pointed at your java.

## Get the scripts

    git clone git@github.com:couchbaselabs/couchbase-ami.git
    cd couchbase-ami

## Setup credentials

    mkdir -p ~/.ec2/couchbase_aws-marketplace

Get the pk/cert for the marketplace-related AWS account.  They'll need to live at...

    ~/.ec2/couchbase_aws-marketplace/pk-RPGT6DCSVXNK5QWMHAACI3KUHN5ILKOX.pem
    ~/.ec2/couchbase_aws-marketplace/cert-RPGT6DCSVXNK5QWMHAACI3KUHN5ILKOX.pem

If your private keys and certs are in a different place, you can
override them by specifying them as KEY=value parameters to the make
command...

    make EC2_PRIVATE_KEY=MyLocationToPrivateKeyPEMFile \
         EC2_CERT=MyLocationToPrivateKeyPEMFile \
         clean

Get your ssh key so you can login into the EC2 instances.  These
usually will live in the ~/.ssh directory on your computer.  For example, mine is at...

    ~/.ssh/steveyen-key2.pem

If you do not have the key yet, using the next step to generate one, and rename it .pem. Make sure the owner can read and write using the key. 

	chmod 600 steveyen-key2.pem

If you don't have an AMI compatible ssh key, run the following command to generate a new one

    make SSH_KEY=steveyen-key2 generate-key

# Building the AMI...

First, clean up from previous attempts...

    make clean

Then, if you're making an AMI for a version number update, be sure to
have the right tag. The common available tags to configure are

	CB_VERSION = <couchbase server version>
	CB_Edition = <couchbase server edition, enterprise/community>
	SYNC_Edition = <sync-gateway edition, enterprise/community>
	CB = <0/1, flag 1 would include CB server, 0 would not>
	
Be aware that you need to manually change SYNC_GATEWAY_URL in Makefile and its downloaded file name as the url changes between versions. SYNC_Edition option only updates AMI description.

First, use step 0, which should launch an new EC2 instance.

    make SSH_KEY=steveyen-key2 step0

If that takes longer than usual (because EC2 cloud is impacted), then repeat the following command untill you finally see some ec2-xxxxxx.compute-1.amazonaws.com addresses in the output...

    make SSH_KEY=steveyen-key2 instance-describe

You'll want to see output lines that look like...

    INSTANCE	i-936991f0	ami-7341831a	ec2-107-22-35-176.compute-1.amazonaws.com	ip-10-93-70-157.ec2.internal	running	steveyen-key2	0		m1.xlarge	2011-10-26T22:59:43+0000	us-east-1c	aki-825ea7eb			monitoring-disabled	107.22.35.176	10.93.70.157			ebs					paravirtual	xen		sg-dddbcdb4	default

Then, go to the next step...

    make SSH_KEY=steveyen-key2 step1

The previous might fail due to SSH issues.  Have patience, wait and try again a few time, as the instance requires time to come online.

Also, you could not create more than 1 AMI image at the same time. The script grabs all the ec2 hostname, so leaving more than 1 ec2 alive could cause connection issue.

If you only have 1 ec2 host alive, and you are still seeing the permission issue, try running "instance-describe" cmd again to  create a new "instance-describe.out".

Then, go to the next step, etc...

    make SSH_KEY=steveyen-key2 step2
    make SSH_KEY=steveyen-key2 step3
    make SSH_KEY=steveyen-key2 step4

By default, couchbase 2.5.1 and sync gateway 1.0 will be installed. Provide tags to override this option.

    make CB_VERSION=2.2.0 CB_Edition=Community SYNC_Edition=Community CB=1 SSH_KEY=steveyen-key2 step2
    make CB_VERSION=2.2.0 CB_Edition=Community SYNC_Edition=Community CB=1 SSH_KEY=steveyen-key2 step3
    make CB_VERSION=2.2.0 CB_Edition=Community SYNC_Edition=Community CB=1 SSH_KEY=steveyen-key2 step4

NOTE: If you don't want the package pre-installed on the AMI, such as to just get an empty-but-ready AMI for QE/testing,  or an AMI only with syn gateway, then just skip step2.

	make SYNC_Edition=Enterprise CB=0 SSH_KEY=steveyen-key2 step1
	make SYNC_Edition=Enterprise CB=0 SSH_KEY=steveyen-key2 step2
	make SYNC_Edition=Enterprise CB=0 SSH_KEY=steveyen-key2 step3

Go to AMI on AWS dashboard. Search for the AMI image number which you just generated on step4. You should now have an AMI that's AWS / ISV Marketplace ready.  But, it might take a few minutes for AWS to finish building it (moving it
out of 'pending' state -- have patience).

# Creating SnapShot

Finally, for an AMI meant for the AWS / ISV Marketplace, Click Action - Modify image permission 
grant permission for AWS to access it...

    grant access to aws # 679593333241

Check the box to, Add "create volume" permissions to the following associated snapshots when creating permissions
Click "Save". You should see the account number listed under "AWS Account Number"

To verify that the snapshot(s) have been correctly shared:
Click "Snapshots" in the left hand navigation bar
Select the snapshot(s) associated with the AMI
Click "Permissions". You should see "aws-marketplace" 


# Other Hints:

If you're doing an updated AMI due to a new software release, be sure to scrub any README's for changes, etc.
To make a community edition AMI, use something like...

    make IMAGE_DESC="pre-installed Couchbase Server 2.0.0, Community Edition, 64bit" \
         PKG_NAME=couchbase-server-community_x86_64_2.0.0.rpm \
         SSH_KEY=steveyen-key2 \
         clean step0 step1 ...
