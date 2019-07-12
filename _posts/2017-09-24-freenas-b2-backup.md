---
title: 'Backing up FreeNAS to Backblaze B2'
date: '27-09-2017 19:09'
categories:
  - 'Blog'
tag:
  - 'FreeNAS'
  - 'Homelab'
---

If you use FreeNAS, it's probably because you care about your data. Part of data security is ensuring the availability of your data. To that end, you need to ensure that said data is backed up. There are generally two reasonable ways to backup your data from FreeNAS. One, local backup (using ZFS replication), and two, cloud backup.

In this article, we will look at setting up cloud backups to Backblaze B2, an economical cloud backup solution similar to Amazon S3.

## Step 1: Sign up

Sign up for a Backblaze account [here](https://www.backblaze.com/b2/sign-up.html). Once you have created an account, go to the "My Settings" tab, and under "Enabled Products", check the box beside B2 Cloud Storage. This enables your account for using B2.

## Step 2: Create a Bucket

Once you have enabled your account for B2, you need to create a bucket (where your files are stored). To do this, on the left side of the screen, select "Buckets" under B2 Cloud Storage. Then, select Create a Bucket. Also on this page, be sure to click on "Show Account ID and Application Key", and mark down your Account ID and click "Create Application Key". Also mark this down, as you will not be able to see it again, and will need it when we setup rclone in a later step.

### Step 2a: Setup Caps and Alerts

While this step is optional, it is highly recommended so that you get notified about any charges against your account that you may not be expecting. I set mine to a cap of $1 a day for each section. This will give you 6Tb of storage, and a good number of API calls.

### Step 2b: Have a look around your account

Have a look around your Backblaze account, there is a great get started guide available [here](https://www.backblaze.com/b2/docs/).

## Step 3: Setting up a FreeBSD Jail on FreeNAS

1. Login to your FreeNAS GUI, and go to the Jails section.
2. Click "Add Jail".
3. Enter a Jail Name. I called mine "b2-backups".
4. Click Ok, and your jail will be created. Note that this may take a little bit of time. You should be able to close the dialog box if needed, the jail will be created in the background.
5. Click on your Jail and click the "shell" button in the bottom left. This will open a shell session to the Jail.
6. Enter `vi /etc./rc.conf` and change `sshd_enable="NO"` to `sshd_enable="YES"`. This will enable SSH to the jail.

! FreeBSD uses vim as a text editor, use `i` to insert text, `del` to delete the rest of the line, and the arrow keys to scroll. Save and exit by pressing the `ESC` key and then `:we` to save and quit.

! You will need to run `passwd root` and reboot in order to have SSH access, as well as `PermitRootLogin yes` in `/etc/ssh/sshd_config`.

!!!! At this point, you can switch over to SSH, if you prefer that to the shell in the FreeNAS GUI.

7. Install `wget` using `pkg install wget`, this will allow you to download the rclone binary.
8. Download the latest rclone binary: `cd /tmp && wget https://downloads.rclone.org/rclone-v1.37-freebsd-amd64.zip`.
9. run `unzip rclone-v1.37-freebsd-amd64.zip` to extract the binary.
!!!! rclone version 1.37 is the latest stable release at the time of this writing
10. Copy the rclone executable to `/usr/bin` by running `cd rclone-v1.37-freebsd-amd64 && cp ./rclone /usr/bin`

### Step 3a: Adding storage to the Jail

1. Create a new folder structure in the Jail, I put mine in `/mnt/storage`, where you will mount your FreeNAS datastores. It is a good idea to make a folder for each dataset you want to mount.
2. In the FreeNAS GUI, go to the Jails tab, and then the Storage sub-tab.
3. Click "Add Storage"
4. Select the Jail you want to add the storage to.
5. Select the source dataset.
6. Select the destination (this will be the folder structure in the jail that you created in Step 3a-1).
7. Optionally select read-only.
8. Leave "Create Directory" selected.
9. Click "Ok".
10. Repeat steps 3a-4 to 3a-9 for each dataset you want to backup to B2.

## Step 4: Configuring rclone

1. Run `rclone config` to initiate the configuration of rclone
2. Press `n` to create a new remote (a remote is what rclone uses to know where to copy/sync your files).
3. Enter a name, I choose `b2`.
4. Press `3`.
5. Enter your account ID from your B2 account.
6. Enter your application Key from your B2 account.
7. Leave endpoint blank.
8. Press `y` to save the config.

### Step 4a: Configuring encryption

1. Follow steps 1-3 from Step 4.
! Note, name this new remote different than the previous remote.
2. Press `5`.
2. Enter the name of the remote you created in Step 4, number 3, followed by the name of your bucket. For example, `b2:storage` in my case.
3. Choose whether or not you want to encrypt the file names, selecting `1` does not encrypt file names. Selecting `2` encrypts the file names. I choose `2`.
4. Choose `y` to type in your own password, choose `g` to generate a strong password randomly. If you choose `g`, you are given an option as to how strong of a password you want to generate.
5. Create a password for the salt. This is recommended if you have chosen to enter your own password in the previous section. Note that for security, these passwords should not be the same.
6. Select `y` to accept the configuration.

!! Note: The rclone config file is not encrypted by default, and Application Keys and your encryption passwords are stored in plaintext. It is recommended to set a password for the config file, and/or ensure the security of the rclone.conf file.

!! If you need to recover encrypted files from B2, you NEED both passwords (if you set two), otherwise your files will be completely unrecoverable.

### Step 4b: Creating the bash script

In this section, we will look at creating the bash script we will use with cron in order to backup any changes to our local storage to B2.

1. Create a new file in `/root`, I called mine `rclone-cron.sh`.
2. Copy the following:

`!/bin/sh
if pidof -o %PPID -x "rclone-cron.sh"; then
exit 1
fi
echo starting storage sync
rclone copy {/path/to/local/storage} {name of your crypt remote}: -v --log-file={/path/to/log/file} --min-age 15m --copy-links
exit`

Let's break that down a bit and look at what the script actually does.

`!/bin/sh` - run the script with the `sh` terminal.

`if pidof -o %PPID -x "rclone-cron.sh"; then` - If the script is currently being run, then:

`exit 1` - Do not run the script currently. This is good if your initial backup will take a while to run, as it won't try to run rclone again.

`fi` - closes the if statement.

`echo start storage sync` - print to the terminal that the clone is starting.

`rclone copy {/path/to/local/storage} {name of your crypt remote}: -v --log-file={/path/to/log/file} --min-age 15m --copy-links` - runs `rclone` with the copy parameter (does not delete files deleted locally, alternatively change `copy` to `sync` to keep an exact copy on B2 (deletes files from B2 that are deleted locally)). Uses the `-v` flag for verbosity. `--log-file={/path/to/log/file}` Tells rclone where to create a log file. `--min-age 15m` Tells rclone not to sync files less than 15 minutes old, useful to ensure copied files are probably complete, instead of semi-completed. `--copy-links` Tells rclone to follow slinks.

`exit` - exits the script when the copy is finished.

### Step 4c: Creating the cron entry

1. Run `crontab -e` to open the cron editor.
2. Enter `0 1 * * * /root/rclone-cron.sh` - This will run the script we created in 4b once a day.

## Step 5: Run the script!

1. `chmod +x /root/rclone-cron.sh` - makes the script executable
2. `cd /root/ && ./rclone-cron.sh` - runs the script.
! rclone does not run in the background. It is recommended to run the script in tmux or similar, or wait for the crontab to run, as the initial backup will probably take a long time if you have a lot of data like I do.

### Step 5a: Check B2 console to see if it's working.

1. Log into your back blaze account, and take a look at your bucket. You should see that files are being copied to B2.

This completes the guide on setting up rclone to backup to B2 on FreeNAS. Rclone can backup to many cloud providers, have a look at different providers if Backblaze is not your cup of tea.