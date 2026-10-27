Building My Splunk Enterprise Servers on AWS (IDX, DS, SH)
This is how I built my three Splunk Enterprise servers on AWS EC2 (RHEL): an Indexer (IDX), a Deployment Server (DS), and a Search Head (SH). I followed the same build on all three. The only per-server differences are the server name and host name.

The commands below are the correct versions of what I ran. I've noted the mistakes I made along the way and how I fixed them, because they were the most useful part of the learning.

I replaced IP addresses, network ranges, and passwords with placeholders: <my-ip> (my home IP), <server-public-ip> (EC2 public IP), <vpc-cidr> (my VPC private range), and <StrongPassword> (Splunk admin password).

Environment
Item	Value
Cloud	AWS EC2, region us-east-2
OS	Red Hat Enterprise Linux
Splunk	Splunk Enterprise 10.6.1.1 (RPM)
Install path	/opt/splunk
Service	systemd (Splunkd.service), runs as the splunk user
SSH client	MobaXterm, logging in as ec2-user
Server	serverName	host
Indexer	IDX	IDX
Deployment Server	DS	DS
Search Head	SH	SH
1. Connect and become root
I connected with MobaXterm as ec2-user, then switched to root:

bash
sudo su - root
Already on the server from earlier setup:

my personal user katryna, a member of the admin group

a logreader group, used to give Splunk read access to /var/log

2. Download and install Splunk
bash
cd /tmp
wget -O splunk-10.6.1.1-b2ae2962645c.x86_64.rpm "https://download.splunk.com/products/splunk/releases/10.6.1.1/linux/splunk-10.6.1.1-b2ae2962645c.x86_64.rpm"
rpm -ivh splunk-10.6.1.1-b2ae2962645c.x86_64.rpm
What I saw during the install:

Header V4 RSA/SHA256 Signature, key ID b3cd4420: NOKEY. This just means Splunk's GPG key isn't imported. The package still installs.

useradd: cannot create directory /opt/splunk. The RPM creates the splunk user before /opt/splunk exists. This is harmless: the user is still created, and the RPM then creates /opt/splunk.

To confirm the splunk user exists:

bash
id splunk
3. Enable boot-start under systemd
bash
cd /opt/splunk/bin
./splunk enable boot-start -systemd-managed 1 -user splunk --accept-license --auto-ports --no-prompt
Result:

Systemd unit file installed at /etc/systemd/system/Splunkd.service

IMPORTANT: Because an admin password was not provided, the admin user will not be created. I handle this with user-seed.conf in step 6.

If you run this command again, Splunk reports that the unit file already exists. That's expected. Run ./splunk disable boot-start first only if you really need to re-create it.

4. Set Linux passwords (optional)
bash
passwd katryna
passwd splunk
katryna is my personal login.

splunk is a service account and doesn't strictly need a password. I set one so I could use sudo as that user (step 5).

Use at least 8 characters. My first attempt warned BAD PASSWORD: The password is shorter than 8 characters, and another failed with Sorry, passwords do not match.

5. Give the splunk user sudo access
I edited sudoers as root, always with visudo, which checks syntax before saving:

bash
visudo
I added this line:

text
%splunk  ALL=(ALL)       ALL
My mistake: I first typed %splunk ALL(=ALL) ALL, which caused:

text
/etc/sudoers:109:13: syntax error
The correct syntax is ALL=(ALL), with the = before the parentheses. At the What now? prompt I chose e to edit again. Never choose Q, because saving a broken sudoers file can lock you out of sudo.

Note: giving a service account full sudo is fine for a lab. In production, I would keep splunk without sudo and do admin work from my own user.

6. Create the Splunk admin user (user-seed.conf)
As the splunk user:

bash
sudo su - splunk
sudo vi /opt/splunk/etc/system/local/user-seed.conf
text
[user_info]
USERNAME = admin
PASSWORD = <StrongPassword>
The password must be at least 8 characters, or Splunk won't create the user. You'll then get "No users exist" on the login page. I documented that problem separately.

Splunk reads this file only on first start, and only if /opt/splunk/etc/passwd doesn't exist. After it creates the user, it deletes user-seed.conf.

7. Base configuration
All files are in /opt/splunk/etc/system/local/. Replace SH with IDX or DS on the other servers.

server.conf
bash
sudo vi /opt/splunk/etc/system/local/server.conf
text
[general]
serverName = SH

[diskUsage]
minFreeSpace = 500
minFreeSpace is in MB. The default is 5000 MB, and Splunk pauses indexing below that. I lowered it to 500 because my lab disks are small. In production, give the server enough disk instead.

inputs.conf
bash
sudo vi /opt/splunk/etc/system/local/inputs.conf
text
[default]
host = SH
I first wrote just host = SH without a stanza header. Splunk treats settings above any stanza as [default], so it works, but writing [default] makes it explicit.

web.conf
bash
sudo vi /opt/splunk/etc/system/local/web.conf
text
[settings]
enableSplunkWebSSL = 1
This turns on HTTPS for the web UI, which is then at https://<server-public-ip>:8000.

8. Start Splunk and fix ownership
Because boot-start is systemd-managed, the cleanest way to start Splunk is with systemd:

bash
sudo systemctl start Splunkd
sudo systemctl status Splunkd
My mistake: I started it with sudo ./splunk restart. That runs the command as root, and anything Splunk creates on first start (var/lib/splunk, etc/auth certs, var/run/...) can end up owned by root. I then ran chown splunk:splunk /opt/splunk, but without -R, so it only fixed the top folder.

The fix:

bash
sudo systemctl stop Splunkd        # or: sudo /opt/splunk/bin/splunk stop
sudo chown -R splunk:splunk /opt/splunk
sudo systemctl start Splunkd
Verification:

bash
ps -o user=,cmd= -p $(pgrep -o splunkd)      # should show: splunk ...
sudo find /opt/splunk ! -user splunk | head  # should print nothing
ls -l /opt/splunk/etc/passwd                 # admin user was created
9. Let Splunk read /var/log
I added the splunk user to the logreader group:

bash
sudo usermod -aG logreader splunk
id splunk
# groups=1004(splunk),1001(logreader)
Restart Splunk afterward so the process picks up the new group:

bash
sudo systemctl restart Splunkd
What I did and what I should have done
I ran:

bash
chown root:logreader /var/log/*
There were two problems:

Changing the group doesn't grant read access. Files like messages, secure, and cron are -rw-------, so the logreader group still can't read them.

It broke ownership that some services need. /var/log/chrony (was chrony:chrony), /var/log/sssd (was sssd:sssd), and btmp, wtmp, and lastlog (were group utmp) all changed. Those services may fail to write their logs.

I restored the original ownership:

bash
chown chrony:chrony /var/log/chrony
chown sssd:sssd /var/log/sssd
chown root:utmp /var/log/btmp /var/log/wtmp /var/log/lastlog
chgrp root /var/log/*.log /var/log/hawkey.log-* /var/log/messages* /var/log/secure* \
          /var/log/cron* /var/log/maillog* /var/log/spooler* /var/log/audit \
          /var/log/insights-client /var/log/private /var/log/rhsm /var/log/tuned
chown root:adm /var/log/cloud-init-output.log   # run last, since *.log above matches it
The better way is ACLs, which give the group read access without changing owners. A default ACL also covers files created later, for example by log rotation:

bash
setfacl -R -m g:logreader:rX /var/log
setfacl -R -d -m g:logreader:rX /var/log
Verification:

bash
getfacl /var/log/messages
sudo -u splunk head -1 /var/log/messages
10. AWS Security Group
All three servers share one security group (ID not shown). Only my IP can reach SSH and the web UI. The Splunk ports are open only inside the VPC.

Port	Purpose	Source
22	SSH	<my-ip>/32
8000	Splunk Web	<my-ip>/32
8089	Management / REST	<vpc-cidr>
9997	Forwarder data (to IDX)	<vpc-cidr>
8088	HEC	<vpc-cidr>
8191	KV Store	<vpc-cidr>
5514	Syslog	<vpc-cidr>
8080	Replication	<vpc-cidr>
None of these rules use 0.0.0.0/0.

Final checklist (per server)
Splunk installed in /opt/splunk, and id splunk works

Splunkd.service exists, and Splunk runs as splunk

sudo find /opt/splunk ! -user splunk prints nothing

/opt/splunk/etc/passwd exists, and I can log in as admin

server.conf has the correct serverName (IDX / DS / SH)

inputs.conf has the correct host

Web UI loads over https://...:8000

splunk is in logreader and can read /var/log/messages

Security group rules match the table above

Lessons learned
Use visudo, and read the error. ALL(=ALL) is wrong. The correct form is ALL=(ALL). Never save a sudoers file with errors.

Don't start Splunk as root. Use systemctl when boot-start is systemd-managed, and use chown -R if root ever touches /opt/splunk.

Group ownership isn't read permission. Use setfacl to give the Splunk user access to logs, instead of chown on /var/log/*.

Wildcard chown is risky. It changed ownership on service directories like chrony and sssd.

Splunk passwords need 8+ characters, or the admin user won't be created.

Watch for paste glitches. A ^[[200~ at the start of a pasted command (MobaXterm bracketed paste) gives command not found. Re-type or re-paste the command.

Keep secrets out of GitHub. Use placeholders for IPs, hostnames, and passwords.
