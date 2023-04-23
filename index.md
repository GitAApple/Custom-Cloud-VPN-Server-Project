

In this installation tutorial, we will be using Digital Ocean to deploy two different servers: Certificate Authority (CA) Server & OpenVPN Server

#### Pre-requisite: Setting up server(s) on DigitalOcean
   
   -  Log in to your Digital Ocean
   -  Click on the "Create" button and then select "Droplets"
   -  Select the region that closest to your location
   -  Select the OS system and version: Ubuntu Version 22.04 (LTS) x64
   -  For other settings, you can choose whatever applicable to you. However, this tutorial is using the following: 
      - Droplet Type: Basic
      - CPU options: Premium Intel; 1 GB/1CPU; 25 GB SSD; 1000 GB tansfer
      - Password Authentication
      - Name your server, so you easily identify which one is which
    - Once everything is configured, click on "Create Droplet" to create your server


#### Step #1 Deploy a Server For OpenVPN
##### 1.1 Configure your OpenVPN server
- Follow the pre-requisite steps, create your Server
- We are going to use SSH to connect your VPN server:
   ```
   ssh root@your_ip_address
   ```
- Create a new non-root admin user
   ```adduser user_name
      usermod -aG sudo user_name
      su user_name
   ```
- Once your are under the non-root user, then you can start updateing your server before other steps by using the following command:
   ```
   sudo apt update && sudo apt upgrade
   ```
- Setting up firewall
   ```
   sudo ufw allow OpenSSH
   sudo ufw enable
   ```

#### Step #2 Deploy a Certificate Authority (CA) Server


##### 2.1 Configure your CA server
-  Follow the pre-requisite steps, create your CA Server
-  Access Console via DigitalOcean
-  Create a new non-root admin user
   ```
   adduser user_name
   usermod -aG sudo user_name
   su user_name```
- Once your are under the non-root user, then you can start updateing your CA server before other steps by using the following command:
   ```
   sudo apt update && sudo apt upgrade
   ```


##### 2.2 Setting up the CA server
- [ ] The first step is to install Easy-RSA, a CA management tool that used to generate private key and root certificate. Enter the following command to install Easy-RSA
   ```
   sudo apt install easy-rsa
   ```

##### 2.3 Preparing a Public Key Infrastructure Directory
   ```
   mkdir ~/easy-rsa
   ```
- Create symbolic links to the easy-rsa pkg installed in step 1.2
   ```
   ln -s /usr/share/easy-rsa/* ~/easy-rsa/
   ```
- Restrict access to PKI directory
   ```
   chmod 700 /home/your_username/easy-rsa
   ```
- Initialize PKI
   ```
   cd ~/easy-rsa
   ./easyrsa init-pki
   ```
- Create a certificate authority
   ```
   cd ~/easy-rsa
   nano vars
   ```
  - Copy and edit following to your vars file, and then Crtl+X, Y, Enter to save
   ```
   ~/easy-rsa/vars
   set_var EASYRSA_REQ_COUNTRY    "US"
   set_var EASYRSA_REQ_PROVINCE   "Hawaii"
   set_var EASYRSA_REQ_CITY       "Honolulu"
   set_var EASYRSA_REQ_ORG        "ITM684 CA Allen"
   set_var EASYRSA_REQ_EMAIL      "admin@ITM684.com"
   set_var EASYRSA_REQ_OU         "Community"
   set_var EASYRSA_ALGO           "ec"
   set_var EASYRSA_DIGEST         "sha512"
   ```

- Build CA
  ```
  ./easyrsa build-ca
  ```
  - New CA Key Passphrase: your_password
  - Enter Common Name or press enter to use default name
  - Note: If you don't want to enter password every time you interact with CA, use this 
  ```
   ./easyrsa build-ca nopass
  ```

##### 2.4 Distrubuting CA's public certificate
  ```
   cat ~/easy-rsa/pki/ca.crt
  ```
- copy all contents within ca.crt and paste it to your OpenVPN server:
  ```
  nano /tmp/ca.crt
  sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/
  sudo update-ca-certificates
  ```

#### Step 3 Install OpenVPN and Easy-RSA on VPN Server Created From Step 1

- Install OpenVPN & easy-rsa under non-root user

  ```
  sudo apt install openvpn easy-rsa
  ```
- Set up and configure easy-rsa directory
 ```
 mkdir ~/easy-rsa
 ln -s /usr/share/easy-rsa/* ~/easy-rsa/
 sudo chown user ~/easy-rsa
 chmod 700 ~/easy-rsa
 ```
#### Step 3 Create Public Key Infrastructure for OpenVPN

- Create and configure PKI
 ```
 cd ~/easy-rsa
 nano vars
 ```
- Insert the following:
 ``` 
 set_var EASYRSA_ALGO "ec"
 set_var EASYRSA_DIGEST "sha512"
 ```
- Initiate PKI
 ```
 cd ~/easy-rsa
 ./easyrsa init-pki
 ```

#### Step 4 Create OpenVPN Certificate request & Private Key

- Navigate to easy-rsa folder under your non-root user and generate certificate request
 ```
 cd ~/easy-rsa
 ./easyrsa gen-req server nopass`
 ```
 - use default common name by pressing enter key or insert your own common name
  - copy generated server key to /etc/openvpn/server
   ```
   sudo cp /home/user/easy-rsa/pki/private/server.key /etc/openvpn/server/
   ```
#### Step 5 Sign your OpenVPN's Ceritificate Request

- Under your OpenVPN Server:
      ```
      scp /home/user/easy-rsa/pki/reqs/server.req sammy@your_ca_server_ip:/tmp
      ```

- Under you CA server as non-root user: Import Certificate Request
 ```
 cd ~/easy-rsa
 ./easyrsa import-req /tmp/server.req server
 ```
- Sign Request
 ```
 ./easyrsa sign-req server server
 ```
- Move both pki/issued/server.crt & pki/ca.crt to your OpenVPN Server /tmp folder using scp.
   ```
   scp pki/issued/server.crt sammy@your_vpn_server_ip:/tmp
   scp pki/ca.crt sammy@your_vpn_server_ip:/tmp
   ```
- Then back to your VPN server and copy files to /etc/openvpn/server folder
 ```
 sudo cp /tmp/{server.crt,ca.crt} /etc/openvpn/server
 ```
#### Step 6 Configure VPN Crptographic
 ```
 cd ~/easy-rsa
openvpn --genkey secret ta.key
sudo cp ta.key /etc/openvpn/server
```
#### Step 7 Generating Client Certificate and Key Pair
-  Setup ClientKey
  ```
  cd ~
  mkdir -p ~/client-configs/keys
  chmod -R 700 ~/client-configs
  cd ~/easy-rsa
  ./easyrsa gen-req client1 nopass
  cp pki/private/client1.key ~/client-configs/keys/
  scp pki/reqs/client1.req sammy@your_ca_server_ip:/tmp
  ```
- Sign Client Key on CA server
  ```
  cd ~/easy-rsa
  ./easyrsa import-req /tmp/client1.req client1
  ./easyrsa sign-req client client1
  ```
- Transfer signed client key back to VPN server:
  ```
  scp pki/issued/client1.crt sammy@your_server_ip:/tmp
  ```
- Under VPN server, move files and configure folder permission
  ```
  cp /tmp/client1.crt ~/client-configs/keys/
  cp ~/easy-rsa/ta.key ~/client-configs/keys/
  sudo cp /etc/openvpn/server/ca.crt ~/client-configs/keys/
  sudo chown sammy.sammy ~/client-configs/keys/*
  ```

#### Step 8 Configuring OpenVPN
  ```
  sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/`
  sudo nano /etc/openvpn/server/server.conf
  ```
- Change conf file with following sections:
  ```
   ;tls-auth ta.key 0 # This file is secret
   tls-crypt ta.key
   ;cipher AES-256-CBC
   cipher AES-256-GCM
   auth SHA256
   ;dh dh2048.pem
   dh none
   user nobody
   group nogroup
   push "redirect-gateway def1 bypass-dhcp"
   push "dhcp-option DNS 208.67.222.222"
   push "dhcp-option DNS 208.67.220.220"
   # Optional!  port 443
   # Optional!  proto tcp
   # if using tcp   explicit-exit-notify 0
  ```


#### Step 9 Adjust VPN Server Networking Configuration
  ```
  sudo nano /etc/sysctl.conf
  ```
- Change the following sections:
  - add this to the bottom of the file: 
   ```
   net.ipv4.ip_forward = 1
   ```
  - then
   ```
   sudo sysctl -p
   ```
#### Step 10 Configure Firewall
-  Find public network interface by using this 
  - `ip route list default`
-  `sudo nano /etc/ufw/before.rules`
- Add these lines to the before.rules file
   ```
   # START OPENVPN RULES`
   # NAT table rules
   *nat
   :POSTROUTING ACCEPT [0:0]
   # Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
   -A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
   COMMIT
   # END OPENVPN RULES
   ```
- Configure UFW
 'sudo nano /etc/default/ufw'
 DEFAULT_FORWARD_POLICY="ACCEPT"
 ```
 sudo ufw allow 443/tcp
 sudo ufw disable
 sudo ufw enable
 ```

#### Step 11 Start VPN Service
 ```
 sudo systemctl -f enable openvpn-server@server.service
 sudo systemctl start openvpn-server@server.service
 sudo systemctl status openvpn-server@server.service
 ```

#### Step 12 Create VPN Client Configuration Infrastructure

mkdir -p ~/client-configs/files`
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf`
nano ~/client-configs/base.conf
- update the following: 
   ```
   remote my_server_ip 443
   proto tcp
   user nobody
   group nogroup
   ```
- comment out `ca`, `cert`, and `key`
- uncomment `tls-auth directive: ;tls-auth ta.key 1`
- update :`cipher AES-256-GCM   auth SHA256`
- add 
   ```
   key-direction 1
   ; script-security 2
   ; up /etc/openvpn/update-resolv-conf
   ; down /etc/openvpn/update-resolv-conf
   ; script-security 2
   ; up /etc/openvpn/update-systemd-resolved
   ; down /etc/openvpn/update-systemd-resolved
   ; down-pre
   ; dhcp-option DOMAIN-ROUTE
   ```
- Create make config bash script
   ```
   nano ~/client-configs/make_config.sh
   ```
  - Copy the bash script content referenced in Step 11 of https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-22-04
   ```
   chmod 700 ~/client-configs/make_config.sh
   ```


#### Step 13 Create Client Config Files
   ```
   cd ~/client-configs
   ./make_config.sh client1
   ```
- This will create a file named client1.ovpn in your ~/client-config/files
- Transfer this file to the device you plan to use as the client. Ex. Cyberduck

#### Step 14 Connect OpenVPN on your client
- Depending on your preference, you can use OpenVPN Connect or Tunnelblick to connect your OpenVPN with the client config file you downloaded from step 13.

#### Step #15 Install Wireguard and IPSec via AlgoVPN scripts

In this step, we are going to install a different VPN called AlgoVPN, which we can use to test the difference between OpenVPN and AlgoVPN. You can find the additional documents for AlgoVPN here: https://github.com/trailofbits/algo 

- Download Algo files from github
  ```
  git clone https://github.com/trailofbits/algo.git
  ```
- Install Algo's core dependencies
  ```
  sudo apt install -y --no-install-recommends python3-virtualenv
  ```
  - Under Algo directory: 
   ```
   python3 -m virtualenv --python="$(command -v python3)" .env && source .env/bin/activate && python3 -m pip install -U pip virtualenv &&  python3 -m pip install -r requirements.txt
   ```
    - `nano config.cfg`
    - depending on your needs, update users section to specify your user. Once completed, save & close
    - Make sure current directory still under algo
    - `./algo`
    - Follow the instruction prompted during deployment to complete the installation
    - After installation, use sftp tool like cyberduck to retrieve your xxx.conf file under /home/user/algo/configs/yourip/wireguard
    - Lastly, use Wireguard application to open this xxx.conf file to connect your VPN


#### Optional Feature #1 Create Cron-jobs
- `crontab -e`
- add new lines for your cron-jobs, for examples:
   ```
   # This cron job runs every 24 hours at 23:00 to check and update packages
   0 23 * * * sudo apt update
   ```
   ```
   # This cron job runs every 24 hours at 0:00 to grab all failed login attempts and save the log to failure_log.txt
   0 0 * * * grep "authentication failure" /var/log/auth.log >> /home/allen/failure_log.txt
   ```
   
   ```
   # This cron job runs every 24 hours at 0:00 to document the user "sal" activity
   0 0 * * * last | grep "sal" > /path/to/user_activity.log
   ```
  


#### Optional Feature #2 Install Different Shell other than bash
There are many shell options like zsh or fish. In this tutorial, we will install fish.
 ```
 sudo apt install fish
 ```
- To switch to fish shell temporarily: 
   ```
   fish
   ```
- To switch to fish shell permanently: 
   ```
   chsh -s /user/bin/fish
   ```


#### Optional Feature #3 Install SSH & Change SSH Port
   ```
   sudo apt install ssh
   sudo nano /etc/ssh/sshd.config
     # uncomment port 22 and change 22 to your new port #
   sudo /etc/services
     # update ssh to your new port #
   sudo ufw allow ssh
   ```

#### Optional Featue #4 Color Your Shell & Promt
- Under your user home directory
   ```
   nano .bashrc
   ```
- Customize your color by adding the following codes (make changes as needed)
   ```
   orange=$(tput setaf 166);
   aqua=$(tput setaf 14);
   green=$(tput setaf 28);
   white=$(tput setaf 15);
   bold=$(tput bold);
   reset=$(tput sgr0);
   blue=$(tput setaf 39);
     
   PS1="\[${bold}\]\n";
   PS1+="\[${aqua}\]\u";   #Color for the username
   PS1+="\[${green}\] at ";
   PS1+="\[${orange}\]\h"; #Color for the hostname
   PS1+="\[${green}\] in ";
   PS1+="\[${blue}\]\W";   #Color for working directory
   PS1+="\n";
   PS1+="\[${green}\]$ \[${reset}\]"; # Color for $ sign then reset
   export PS1;
   ```


#### Optional Featuer #5 Creating Alias
* Under your user home directory
  * `nano .bashrc`
  * Use `alias` to create your alias,  see below for example
    ``` 
    alias ..='cd ..'
    alias c='clear'
    alias count='find . -type f | wc -l'
    ```
  

#### Problems Occured During This Project:
* Be careful when editing your client config file (Step 12)
  * The instruction may tell you to update with my_server_ip, make sure you change it to your actual server ip address and not the word "my_server_ip".
* After changing your port in Optional #3, you may experience some issues when trying to ssh into your server. Be sure to use the following command as well:
  ```
  sudo /sbin/iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport xxxxx -j ACCEPT
  ```
  * replace xxxxx with your port number
  
