# IT Service Desk Lab (GLPI on Ubuntu Server)

**Setup:**
- Hypervisor: VirtualBox 7.2
- Server: Ubuntu Server 26.04 (3 GB RAM)
- Domain controller: Windows Server 2022 (from the AD lab)
- Client: Windows 11 Enterprise (from the AD lab)
- Web stack: Apache 2.4, MySQL 8.4, PHP 8.5 (PHP-FPM)
- Ticketing: GLPI 11.0.9


**Skills:** 
- Linux server administration (Ubuntu Server)
- Network configuration (Netplan, static IPs, private addressing)
- DNS (forward and reverse lookup zones, A and PTR records)
- Apache (virtual hosts, modules)
- MySQL (secure installation, least-privilege accounts)
- PHP-FPM
- ITSM with GLPI (categories, triage, ticket lifecycle)
- Active Directory (users, OUs, security groups)
- Time synchronisation (chrony)
- Security hardening
- Troubleshooting

## Installing the VM and joining the network

To start with I set the first adapter in VirtualBox to NAT because unlike my previous virtual machines this one needs to connect to the internet to install packages. I set the second adapter to internal so it can communicate with the other machines.

![Ubuntu error](../Images/Ubuntu%20images/Ubuntu-error.png)


After restarting this is the first error I get and pressing enter doesn't seem to give me a fresh login prompt. Looking it up this error seems to be related to VirtualBox's graphics controller so I shutdown the machine and switch from VMSVGA to VBoxVGA. Booting it up again gives me a login screen.

![Ubuntu login](../Images/Ubuntu%20images/Ubuntu-login.png)

#### Fixing Network IPs

Previously I had set my win domain controller's IPv4 to 192.254.83.47 however as I noted at the time that's a public range and best practice would be to use a private range like 192.168.x.x. That didn't matter previously due to the lab only using an internal network however the new ubuntu machine will need to connect to the internet to install packages so this would be a good time to fix it.  Before starting I will make a snapshot of my VM just in case something goes wrong. 

![Domain controller ipconfig output](Pasted%20image%2020260923200403.png)

*Old domain controller ipconfig*

I chose to make the new IP 192.168.50.10 with a 255.255.255.0 subnet since that's a private range and won't cause problems in the future.  Previously I set the default gateway to the loopback address (`127.0.0.1`) but this lab didn't need a default gateway due to being an internal network. On top of that, a default gateway needs to be a router on the same network whereas the loopback address simply points at the machine so I changed it to blank.

![New domain controller ipconfig](new-dc-ipconfig.png)

*New domain controller ipconfig*


![Old client ipconfig](client-old-ipconfig.png)

*Old client ipconfig*

First I made a snapshot same as the other machine. After logging in to the client machine I realised that Kai does not have admin access and since I changed the DC IP the client still had its DNS pointed at the old IP so I couldn't log in to the `HLBholdings\Administrator` account. 

![client-local-admins](client-local-admins.png)

I ran `net localgroup administrators` to check who has admin rights on the machine and realised I still had the vboxuser left on the machine. I logged into the default account and changed the ipv4 to 192.168.50.20 and made sure to update the subnet mask and point DNS to my DC.

![New client ipconfig](new-client-ipconfig.png)

*New client ipconfig*

To test this more thoroughly I tried pinging the DC's IP, domain and doing a nslookup.

![new-client-ipconfig-tests](../Images/Ubuntu%20images/new-client-ipconfig-tests.png)


The two pings came back fine however the nslookup came back with a `Server: UnKnown`. The lookup itself worked as I got the correct answer for the address and name however the server name is missing. This tells me that the nslookup found the DNS server then asked for a PTR record which lives in the reverse lookup zone and timed out. Going to the DC I check the DNS manager and see that the reverse lookup zone is empty.


I made a new reverse lookup zone then ran `ipconfig /registerdns` on both machines. Checking the reverse lookup zone I can see that it now has PTR records for both IPs and running nslookup again on the client confirms it can get server name now. 

![dns-manager-rlz](dns-manager-rlz.png)

![dns-manager-rlz-inside](dns-manager-rlz-inside.png)

![client-nslookup](client-nslookup.png)
#### Setting Ubuntu IP

Now we have the network IPs set I can begin on the ubuntu machine. First I check for the current IP config using `ip a`. 

![ubuntu-ip-config](ubuntu-ip-config.png)

It shows our loopback address, enp0s3 that has a 10.0.2.15 inet which is the NAT adapter and enp0s8 which has no ipv4 address so that's our internal adapter. Next I went to check what's in `/etc/netplan` and found a `00-installer-config.yaml`. Using `cat` I checked the contents and found our NAT settings but no enp0s8 set.

![ubuntu-netplan-startfig](../Images/Ubuntu%20images/ubuntu-netplan-startfig.png)

Before changing the file I will back it up with `sudo cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.bak` so if any mistakes happen I can roll back. Putting `.bak` at the end is to prevent Netplan reading the file as it only reads `.yaml` files.

![ubuntu-netplan-backup](ubuntu-netplan-backup.png)

*Showing the file successfully copied*

Now I can use `nano` to edit the file.

![new-ubuntu-netplan-config](new-ubuntu-netplan-config.png)

After saving the changes I use `sudo netplan try` to apply the changes. Running another `ip a` to check my ipconfig.

![ubuntu-new-ipconfig](ubuntu-new-ipconfig.png)

Looking under enp0s8 you can see the new inet which confirms the changes saved and applied properly.

![Ubuntu ping to the DC and resolvectl status](ubuntu-ping-resolvectl.png)

The ping also shows the ubuntu server can reach the DC however the `resolvectl` shows both DNS servers have default routes which could lead to a query meant for the internet going to the internal DNS and failing. To test this I query the local domain and ubuntu.com to see if both resolve properly.

![resolvectl-DNS-query-check](resolvectl-DNS-query-check.png)

Looking at the results the domain query uses enp0s8(my internal network) and the ubuntu.com uses enp0s3(my NAT) which means both are routing correctly. On the DC side however, the ubuntu server isn't a domain member so the DNS manager wont automatically make a record for it. 

![ubuntu-forward-error](../Images/Ubuntu%20images/ubuntu-reverselookup-error.png)

Using `dig @192.168.50.10 UbuntuServer.HLBholdings.local` to check the record I can see `status: NXDOMAIN` which means the record for the UbuntuServer doesn't exist. This is to be expected since I haven't made the record for it yet in DNS manager.

![ubuntu-dns-record](ubuntu-dns-record.png)

Making the record and calling it `UbuntuServer` if I run the command again it should return `NOERROR`.

![ubuntu-reverselookup-success](../Images/Ubuntu%20images/ubuntu-reverselookup-success.png)

And it does, confirming that the DC now resolves UbuntuServer.HLBholdings.local to 192.168.50.30.

## Installing LAMP stack

Before starting I will create a snapshot on my Ubuntu machine. To refresh the server's list of available packages I ran `sudo apt update` however I got 3 errors saying "is not valid yet" which makes me think we have a time issue. 

![sudo-apt-update](../Images/Ubuntu%20images/sudo-apt-update.png)

Upon running `timedatectl` it shows clock is behind by a day since it's showing the 28th and today is the 29th which is probably due to me saving state rather than shutting down the VM. 

I attempted to restart `systemd-timesyncd` but it didn't have the service which was unusual so I looked it up and apparently Ubuntu 26.04 uses chrony instead.

![](../Images/Ubuntu%20images/systemd-timesyncd-not-found.png)

Checking chrony's status shows it was up the whole time. It probably fell behind when the VM was restored from a saved state where it had already stepped the clock and from there it could only gradually slew the time.

![](../Images/Ubuntu%20images/systemctl-status-chrony.png)

![](../Images/Ubuntu%20images/chronyc-makestep-200.png)

Running `sudo chronyc makestep` which forces chrony to step the clock immediately returned a 200 OK and checking the clock again with `timedatectl` showed that the clock had now synced.

![ubuntu-timedatectl](../Images/Ubuntu%20images/ubuntu-timedatectl.png)

Now `sudo apt update` runs cleanly with no errors.

![sudo-apt-update](../Images/Ubuntu%20images/sudo-apt-update%201.png)

Next I ran `sudo apt upgrade` to bring the installed packages up to date and rebooted.


#### Installing Apache

I'm going to make another snapshot since this clean boot is a fresh state then do `sudo apt install apache2` . The install seemed to go fine so I ran `systemctl status apache2` and it was active.

![systemctl-status-apache2](../Images/Ubuntu%20images/systemctl-status-apache2.png)

At the bottom of the log I can see that Apache "Could not reliably determine the server's fully qualified domain name" which I will fix now for clean logs.

I used `sudo nano /etc/apache2/conf-available/servername.conf` to make a config file and named the server UbuntuServer.HLBholdings.local. 


![Server-Name-apache](../Images/Ubuntu%20images/Server-Name-apache.png)

After enabling config and reloading it the AH00558 warning no longer shows.

![sudo-a2enconf-apache.](../Images/Ubuntu%20images/sudo-a2enconf-apache.png)

Testing both `192.168.50.30` and `ubuntuserver.hlbholdings.local` on the client machine I get the Apache2 default page which means it's working.

![apache2-default-page](../Images/Ubuntu%20images/apache2-default-page.png)

#### Installing MySQL

Next I ran `sudo apt install mysql-server`. It installed fine so I ran `systemctl status mysql` and it was active.

![mysql-server-active](../Images/Ubuntu%20images/mysql-server-active.png)

I originally only gave this VM 2GB of RAM but upon seeing that MySQL was showing 478.5M I decided to check with `free -h` which showed it only had 1.6GB. Out of that only 783MB was left so I bumped the RAM up to 3309MB.

Then I ran `sudo mysql_secure_installation` below are my security choices:
- **Password validation:** on, STRONG
- **Anonymous users:** removed
- **Remote root login:** disabled
- **Test database:** removed
- **Privileges:** reloaded

I generally chose the safest and most secure choices all round. Keeping strong passwords is standard practice. I removed anon users since nothing in this lab requires that so keeping it open is just an open door. For root login I didn't plan on controlling this remotely and if I left it on anyone who could reach port 3306 would be able to attempt a login remotely. The test database was removed so I didn't have an unsecured database in my network that I didn't need.

#### Installing PHP and setting up for GLPI

Before installing GLPI I will prepare the MySQL server. I used `sudo mysql` to get into MySQL as root.

Here is a summary of the commands I used to create the GLPI database and give privileges to the GLPI account so it can only touch the GLPI database and nothing else following the principle of least-privilege.

```
CREATE DATABASE glpi;
CREATE USER 'glpi'@'localhost' IDENTIFIED BY 'YourStrongPasswordHere';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpi'@'localhost';
SHOW DATABASES;
SHOW GRANTS FOR 'glpi'@'localhost';
exit;
```


![sql-databases](../Images/Ubuntu%20images/sql-databases.png)


Next I install PHP-FPM plus GLPI's required and optional PHP extensions. I chose PHP-FPM over mod_php due to the fact it runs more efficiently and it runs PHP as a separate service from Apache so restarting or tuning PHP won't mean restarting Apache.

```sudo apt install php8.5-fpm php8.5-mysql php8.5-curl php8.5-gd php8.5-intl php8.5-xml php8.5-zip php8.5-bz2 php8.5-ldap```


FPM isn't linked to Apache by default so I enable `proxy_fcgi` and `setenvif` with `a2enmod` then enabled the FPM config with `a2enconf` .  After restarting I setup a temporary page to test it's up. 

![php-info-site.](../Images/Ubuntu%20images/php-info-site.png)

I then deleted the page since having a page with all of my version numbers on it can be used by attackers to find exploits.

#### Installing GLPI

I chose to use 11.0.9 (the latest stable release) and downloaded it with `wget` into `/tmp` and extracted with `tar`.

![glpi-download-and-extract](../Images/Ubuntu%20images/glpi-download-and-extract.png)

I moved it to `/var/www/glpi` and gave ownership to `www-data:www-data` (the web server user). The GLPI docs don't recommend you do this as an attacker that exploits a vulnerability in GLPI or a plugin then has their code run as www-data and is allowed to edit GLPI's php files letting them do stuff like plant a backdoor. However for this lab I judged it fine since it's a closed environment and doing it that way could cause perms issues that I would need to debug. In production however I would make it so the only folders GLPI writes to (files, config and marketplace) are owned by www-data and the rest is owned by root like GLPI recommends.

![glpi-chown-and-ls](../Images/Ubuntu%20images/glpi-chown-and-ls.png)

Next I wrote the virtual host file making sure to keep only the web root public so config, uploads and code can't be reached by URL. The rewrite rules make sure requests that aren't real files go to GLPI's `index.php` which handles pages.

![glpi-virtualhost-file](../Images/Ubuntu%20images/glpi-virtualhost-file.png)

I used `a2ensite` to enable the VirtualHost file and ran `a2enmod rewrite` to activate Apache's rewrite module. After disabling Ubuntu's default site (to stop it catching requests meant for GLPI and reducing attack surface) checking with `configtest` and restarting Apache2, I opened `ubuntuserver.hlbholdings.local` on the client machine and can see the GLPI setup page confirming it worked.

![GLPI-setup-page](../Images/Ubuntu%20images/GLPI-setup-page.png)

Going through the install steps I saw I was missing some of the required extensions (mbstring and bcmath) so I went back to the Ubuntu machine to install them.

![glpi-missing-extensions](../Images/Ubuntu%20images/glpi-missing-extensions.png)

After installing them and reloading the website it now listed me as having all the required extensions.

![glpi-extensions-passed](../Images/Ubuntu%20images/glpi-extensions-passed.png)



![glpi-database](../Images/Ubuntu%20images/glpi-database.png)

After selecting the database I set up (glpi) the installation finished and showed me the list of default accounts. 

![glpi-default-accounts](../Images/Ubuntu%20images/glpi-default-accounts.png)

## Configuring GLPI

#### Accounts

The dashboard has a warning on it telling me to change the default accounts passwords. Since the default accounts all have publicly known passwords leaving them alone would be dangerous since anyone with access to the login page could login as admin using the glpi account.

![glpi-dashboard](../Images/Ubuntu%20images/glpi-dashboard.png)

Going to check the accounts they are all general accounts.

![glpi-default-accounts](../Images/Ubuntu%20images/glpi-default-accounts%201.png)

The first thing I do is go and make a new super admin account and log into that so I can go ahead and disable the default accounts. My reasoning for doing this is so I can assign specific employee accounts rather than having them use general ones giving us accountability when tracking which actions are attributed to which person. Doing this will also remove well-known usernames and allow me to practice the principle of least privilege when making the new accounts.

![super-admin-elijah](../Images/Ubuntu%20images/super-admin-elijah.png)

*New super admin*

![glpi-disabled](../Images/Ubuntu%20images/glpi-disabled%20accounts.png)

*default accounts disabled*

glpi-system gets left because it's not a user account instead it's meant to be used by GLPI for automatic actions. On top of that it has no known default password. 


Going back to the dashboard the warning to change account password is gone letting us know that disabling them worked.

![dashboard-warning-cleared](../Images/Ubuntu%20images/dashboard-warning-cleared.png)


I then add some of my employees as accounts. Isabelle being the only other IT team member gets technician meanwhile Fatima and Maya get self service as they will be the ones making the tickets.

![account-types](../Images/Ubuntu%20images/account-types.png)


#### Creating ticket categories and tickets

For this lab I will have 4 ticket types being accounts and access, network and connectivity, printing and software since these generally cover what I can show in a virtual machine lab.

Going to Setup -> Dropdowns -> ITIL categories I make my categories. 

![ticket-categories](../Images/Ubuntu%20images/ticket-categories.png)


I will make 4 tickets with half being real and half being fake:

- Fatima forgets password and requests reset
- Maya loses access to graphics team share folder
- Fatima's prints aren't coming out
- Maya needs Photoshop for an advertising campaign

For the real ones I will first remove Maya from the Graphics team security group then login to her account and confirm she does not have access to the share folder.

![maya-missing-folder](../Images/Ubuntu%20images/maya-missing-folder.png)

Next I will login to Fatima's GLPI and make her 2 tickets. Then I will do the same for Maya.

![tickets](../Images/Ubuntu%20images/tickets.png)

Logging in to GLPI as Isabelle we can see she now has 4 new tickets. In terms of order I will tackle them from highest business impact to lowest which I believe to be.

1. Fatima's sign-in: she can't work at all, so it has the highest impact.
2. Maya's missing folder: This is bad but she still has access to her machine and can do other stuff.
3. The printer: This is annoying but not a huge block
4. The Photoshop request

Now I have assigned all tickets to Isabelle. And we can see in her personal view that all the tickets are there.

![4-unresolved-tickets](../Images/Ubuntu%20images/4-unresolved-tickets.png)


#### Solving the tickets

Usually before issuing a password reset I would try to verify Fatima's identity either with a call or in person ideally but since this is a lab I will just reset her password from the DC using Active Directory.


![fatima-login-reset](../Images/Ubuntu%20images/fatima-login-reset.png)

Now on entering the temp password it prompts her for a new one.

Back on Isabelle I can add a solution before going back to Fatima and approving it.

![fatima-password-solution](../Images/Ubuntu%20images/fatima-password-solution.png)

Since we already know what Maya's problem is I will add her back to the Graphics Team security group on the DC.

![maya-security-group](../Images/Ubuntu%20images/maya-security-group.png)

Now if we go back to Maya's account and sign out and back in we can see she has access to the share again.

![maya-share-folder](../Images/Ubuntu%20images/maya-share-folder.png)

All that's left is to have Isabelle put the solution down and approve it from Maya's GLPI account.

![maya-solution](../Images/Ubuntu%20images/maya-solution.png)

For the next 2 tickets since they are fake I will just go to the relevant accounts and solve them. Now through Isabelle's view we can see she successfully closed all her tickets.

![closed-tickets](../Images/Ubuntu%20images/closed-tickets.png)

## Problems and fixes

| Problem                                                                | Cause                                                                                | Fix                                                                            |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| Ubuntu showed a graphics error at first boot and no login prompt       | The VMSVGA graphics controller didn't work with this guest                           | Switched the controller to VBoxVGA                                             |
| Couldn't use a domain admin account on the client                      | The client's DNS still pointed at the DC's old IP, so it couldn't reach the DC       | Used the local vboxuser admin account to change the client's IP and DNS        |
| `nslookup` showed "Server: UnKnown" and timed out                      | There was no reverse lookup zone, so the DC's IP couldn't be turned back into a name | Created a reverse lookup zone and ran `ipconfig /registerdns` on both machines |
| The DC couldn't resolve the Ubuntu server's name                       | Ubuntu isn't a domain member, so it doesn't register itself in DNS                   | Manually created an A record (and PTR) for UbuntuServer                        |
| `apt update` failed with "not valid yet"                               | The server's clock was a day behind after restoring from a saved state               | Forced chrony to correct the clock with `chronyc makestep`                     |
| Couldn't restart `systemd-timesyncd`                                   | Ubuntu 26.04 uses chrony for time sync instead                                       | Checked and fixed the clock through chrony                                     |
| Apache showed the AH00558 warning                                      | Apache didn't have a server name set                                                 | Added a `ServerName` config and enabled it with `a2enconf`                     |
| Low memory after installing MySQL                                      | 2 GB was too little for the web stack, and there was no swap                         | Raised the VM's RAM to about 3 GB                                              |
| GLPI's installer reported missing extensions                           | mbstring and bcmath weren't installed                                                | Installed both extensions                                                      |
| GLPI's default accounts had public passwords                           | GLPI ships with well-known default logins                                            | Created named accounts and disabled the defaults                               |
| Maya still couldn't see her folder after being added back to the group | Her access token still had her old group memberships                                 | Signed her out and back in to refresh her token                                |