# Using Jenkins to automate nearcore CD 

In this guide I will go over how I set up jenkins to monitor the nearcore github for new updates on specific releases, retrieve the new releases, compile the source code locally, run automated testing, and finally deploy a node using the new software on its own localnet. This should work with a new install of ubuntu 18.04

## Hardware info
I believe the hardware requirment minimum for this project to be a 4 core cpu with 8gb of ram and a 125gb ssd hard drive.  

## Tools required

- Linux host (with 10gb swap)
- Jenkins
- Docker
- Rustup

## Configure Ubuntu

- Install required packages
```
sudo apt install python3 git curl libclang-dev build-essential llvm runc gcc g++ unattended-upgrades make clang pkg-config libssl-dev libudev-dev g++ g++-multilib lib32stdc++6-7-dbg libx32stdc++6-7-dbg cmake openjdk-11-jre-headless openjdk-11-jdk-headless apt-transport-https ca-certificates curl gnupg-agent software-properties-common
```

- Check swapfile
```
sudo swapon --show
NAME      TYPE SIZE  USED PRIO
/swapfile file  10G 21.8M   -2
```

- If you have no swapfile set up use you can create and enable one with these commands
```
sudo fallocate -l 10G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo su
echo '/swapfile swap swap defaults 0 0' >> /etc/fstab
exit
sudo swapon --show
NAME      TYPE SIZE  USED PRIO
/swapfile file  10G 21.8M   -2
```

## Install Docker

[Docker Offical Documentation](https://docs.docker.com/engine/install/) <--- Source

- {Install Option} To deploy from jenkins to another host running docker that host must have docker api on tcp port for jenkins to use. By doing this we are introducing a way into root. This step is **NOT** required for successful build and has been removed from this document. Instructions can be found in offical documentation
- If you do this set a firewall rule to block incoming connections on your internet ethernet interface port 2375 (Get interface name with ifconfig)
```
sudo iptables -I INPUT 1 -i eth0 -p tcp -m tcp --dport 2375 -j DROP
```

[Security Implications of the docker api](https://docs.docker.com/engine/security/)


- Remove anything related to docker and install docker from the docker apt repo
```
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint
sudo apt-key fingerprint 0EBFCD88

pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo groupadd docker
sudo usermod -aG docker <your_user_name>
sudo systemctl enable docker
```

## Install Jenkins

[Jenkins Official Documentation](https://docs.docker.com/engine/install/ubuntu/) <--- Source

- Jenkins requires java to be installed verify your version is 11.0.8

```
java -version
openjdk version "11.0.8" 2020-07-14
```

- There are 3 possible way to install Jenkins on ubumtu. I tried all 3. I recommend installing using apt. 
- First we get the public key for the jenkins apt repository, then add source to the source.list, then update and install.

```
sudo add-apt-repository universe
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
sudo usermod -aG docker jenkins 
sudo systemctl enable jenkins
```
- Retrieve the adming password and save it
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

- Restart the system now with "sudo systemctl reboot -i" 

- If you have a firewall installed *(you should)*, jenkins admin interface listens on port 8080. Allow incoming connections on port 8080
```
sudo iptables -A INPUT -i eth0 -p tcp --sport 8080 -j ACCEPT
```


# Install rustup For Jenkins User

- Log in with the jenkins user & install rustup

```
sudo su jenkins

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# choose option 1
source $HOME/.cargo/env
rustup component add clippy-preview
rustup default nightly
# it should update then return 
# nightly-x86_64-unknown-linux-gnu installed - rustc 1.48.0-nightly (7402a3944 2020-09-13)
source ~/.profile
exit
```


## Usage

We will use the test script start_unittest.py there are more scripts located in the source or https://github.com/nearprotocol/nearcore/tree/master/scripts


- Login with the adming password at http://<server_ip>:8080
- Create a user and install default plugins
- Go to manage jenkins, Add the docker plugin
- Restart jenkins after plugin install
- Make a new Item. Create a new freestyle project 
- Set git as source 
- Repository URL:	```https://github.com/nearprotocol/nearcore.git```
- Click advanced. Refspec: ```+refs/tags/*:refs/remotes/origin/tags/* ```
- Branch Specifier (blank for 'any'): ```*/tags/*1.14.*-beta.*```
- Select Poll SCM schedule: ```H/30 * * * *```
- Create 2 build steps. 
- first = make release
- second = python3 ./scripts/start_unittest.py
- Save the item
      
 # Run
 
 In the project click on build now and it is scheduled to run every 30 minutes
 
 You can look in the console output for any job to view the job log
 
 This should compile 1.14.<any>-beta.3 and run several tests
 
 - At the end of the compile operation you should see this in the console output
```
+ python3 ./scripts/start_unittest.py
[2mSep 13 05:48:31.981[0m [32m INFO[0m near: Version: 1.2.0, Build: 349f73a7, Latest Protocol: 30    
[2mSep 13 05:48:31.982[0m [32m INFO[0m near: Generated node key, validator key, genesis file in /srv/near    
Starting unittest nodes with test.near account and seed key of alice.near
Setting up network configuration.
Stake for user 'test.near' with 'ed25519:22skMptHjFWNyuEWY22ftn2AbLPSYpmYwGJRGwpNHbTV'
Starting NEAR client and Watchtower dockers...
Node is running! 
To check logs call: docker logs --follow nearcore
Finished: SUCCESS
```
- Now you can log into the docker and use the newly created validator / localnet

```
docker exec -it nearcore /bin/bash
near --help
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
