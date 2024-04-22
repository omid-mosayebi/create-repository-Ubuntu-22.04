Setup Local APT Repository Server on Ubuntu 22.04
Prerequisites
•	Pre Installed Ubuntu 22.04
•	Apache Web Server
•	Regular User with sudo rights
•	Minimum of 240 GB free disk space on /var/spool file system
•	A reliable internet connection to download packages
1) Install Apache Web Server
First off, log in to your Ubuntu 22.04 system and set up the Apache web server as shown.
	sudo apt install -y apache2
Apache’s default document root directory is located in the /var/www/html path. We are later going to create a repository directory in this path that will contain the required packages needed.
2) Create Package Repository Directory
Next, we will create a local repository directory called ubuntu in the /var/www/html path.
	sudo mkdir -p /var/www/html/ubuntu
Set the required permissions on above created directory.
	sudo chown www-data:www-data /var/www/html/ubuntu
3) Install Apt-Mirror Utility

The next step is to install apt-mirror package, after installing this package we will get apt-mirror utility which will download and sync the remote Debian packages to our local repository on our server. So for its installation, run
	sudo apt update
sudo apt install -y apt-mirror
4) Configure Apt-Mirror

Once apt-mirror is installed then its configuration file ‘/etc/apt/mirrror.list’ is created automatically. This file contains list of repositories that will be downloaded or sync on the local folder of our Ubuntu server. In our case local folder is ‘/var/www/html/ubuntu/’. Before making changes to this file let’s backup first using cp command.
	sudo cp /etc/apt/mirror.list /etc/apt/mirror.list-bak
Now edit the file using vi editor and update base_path and repositories as shown below.
	sudo vi /etc/apt/mirror.list
	
	############# config ###################
set base_path    /var/www/html/ubuntu
set nthreads     20
set _tilde 0
########## end config ##############
###deb http://archive.ubuntu.com/ubuntu/ jammy main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ jammy main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ jammy-updates main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ jammy-backports main restricted universe multiverse

clean http://archive.ubuntu.com/ubuntu
Save and exit the file.
5) Start Mirroring Repositories to Local Folder
Now, you can run apt-mirror command to start the mirroring process. This will download the packages from the specified repositories and store them in the designated local folder (base_path).
	sudo apt-mirror
The mirroring process may take some time even couple of hours, depending on your internet speed and the size of the repositories.
Above command can also be started in the background using below nohup command,
sudo apt-mirror &
Great, output above confirms that the mirroring is completed.
In Ubuntu 22.04, there are some issue noticed with apt-mirror command like it does not mirror CNF folder, icon tar files and binary-i386, so to fix these issues, create a script with following content and execute it.
$ vi fix-errors.sh
#!/bin/bash

cd /var/www/html/ubuntu/archive.ubuntu.com/ubuntu/dists

for dist in jammy jammy-updates jammy-security jammy-backports; do
  for comp in main multiverse universe; do
    for size in 48 64 128; do
    wget http://archive.ubuntu.com/ubuntu/dists/$dist/$comp/dep11/icons-${size}x${size}@2.tar.gz -O $dist/$comp/dep11/icons-${size}x${size}@2.tar.gz;
   done
 done
done

cd /var/tmp
for p in "${1:-jammy}"{,-{security,updates,backports}}\
/{main,restricted,universe,multiverse};do >&2 echo "${p}"
wget -q -c -r -np -R "index.html*"\
"http://archive.ubuntu.com/ubuntu/dists/${p}/cnf/Commands-amd64.xz"
wget -q -c -r -np -R "index.html*"\
"http://archive.ubuntu.com/ubuntu/dists/${p}/cnf/Commands-i386.xz"
wget -q -c -r -np -R "index.html*" \
"http://archive.ubuntu.com/ubuntu/dists/${p}/binary-i386/"
done

sudo cp -av archive.ubuntu.com/ubuntu/ /var/www/html/ubuntu/archive.ubuntu.com
save and close the file.
sudo chmod +x fix-errors.sh
sudo bash fix-errors.sh
Note: We need to execute the above script only once.
Next create the following symbolic link so that we can access repository over the browser as well.
sudo ln -s /var/spool/apt-mirror/mirror/archive.ubuntu.com /var/www/html/ubuntu

6) Scheduling Mirroring using Cronjob
Configure a cronjob to automatically update our local apt repositories. It is recommended to setup this cron job in the night daily.

Run crontab command and add following command to be executed daily at 1:00 AM in the night.
$ sudo crontab -e

00  01  *  *  *  /usr/bin/apt-mirror
Save and close.
7) Access Local APT Repository Using Web browser
In case Firewall is running on your Ubuntu system then allow port 80 using following command
	$ sudo ufw allow 80
Now, try to access locally configured apt repository using web browser, type the following URL:
http://<Server-IP>/ubuntu/archive.ubuntu.com/ubuntu/dists/
8) Configure Client to Use Local Apt Repository Server
To test and verify whether our apt repository server is working fine or not, I have another Ubuntu 22.04 system where I will update /etc/apt/sources.list file so that apt command points to local repositories instead of remote.

So, login to the system, change the following in the sources.list

deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy main restricted
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy-updates main restricted
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy universe
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy-updates universe
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy multiverse
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy-updates multiverse
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy-security main restricted
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy-security universe
deb [arch=amd64 trusted=yes] http://IP-repository/mirror/archive.ubuntu.com/ubuntu/ jammy-security multiverse

