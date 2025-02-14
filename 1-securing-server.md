# Securing my server:
- I ssh into my server using `ssh root@<ip>` and enter my password when prompted. Now I am logged in as root.
- Here are the things that needs to be done:
    1. I have to update the system.
    2. As a convention according to principle of least privilege you don't stay logged in as root for long and instead use `sudo` to perform root actions. So, I will be creating and admin user.
    3. I wanna setup ssh key pairs to login to my server from my mac without my password. This is more secure and recommended for obvious reasons. Also I have to change some default ssh configurations like port, root login, password login, etc.
    4. I also wanna check my current status of server like the hardware being consumed, processes running, etc.
    5. I also wanna see the current firewall rules and maybe configure them.
    6. I also wanna link my linux to my github account.

# 1. Creating new user with admin privilege for my VM:
- `sudo adduser <username>`
- `sudo usermod -aG sudo <username>`

# 2. Updating and Upgrading the system:
- `sudo apt update && sudo apt upgrade -y`
- `sudo reboot`

# Problem 1: grub-install: error: cannot find EFI directory. Encountered while updating and upgrading linux.
- Displaying currently mounted disks, their mounted path: `df`. Use man to check its option based on the info you want. 'h' a
nd 'a' flag are useful.
- /dev: stores device files, which are special files that represent hardware devices and virtual devices. These files allow software to interact with hardware by reading from and writing to these files. /dev/sda or anything starting with sd under /dev/ usually represents block devices like SSDs, HDD and USB.
- **Diagnosis**: 
    - EFI mode vs BIOS mode: First of all EFI is often called UEFI as well. Don't get confused. There is a lot of information but basically: BIOS is the older, simpler firmware interface, limited by MBR partitioning and a text-based interface. UEFI is the modern, more advanced firmware interface, supporting GPT partitioning, larger disks, and a graphical interface.
    - My system was installed in BIOS mode. How did I figure that out? `/sys/firmware/` doesn't have `efi`. And when I run `sudo efibootmgr` it gives me `EFI variables are not supported on this system.`
    - And upon running this command: `dpkg -l | grep grub`, I found out that I have both UEFI and BIOS version of grub package. One of them being redundant was updated when I ran the update/upgrade command and since it is not really setup, I got the error.
- Possible fix/solution: q`sudo apt remove grub-efi-amd64-bin grub-efi-amd64-signed`. 
- I won't do it though, because I really don't know why Hostinger installed it, this might be dependency for some services they provide for virtual machine.
    - Solution: `sudo apt remove --simulate grub-efi-amd64-bin grub-efi-amd64-signed`: this simulates how the system would react if a package was removed.


# 3: SSH setup.
- So, I performed this setup on my personal computer, which I will be using to connect to my linux.
- `ssh-keygen -t rsa -b 4096 -C "youremail"` -b option specifies the number of bits in the key. 4096 bits is a strong key size, providing a high level of security. The default size is usually 2048 bits. youremail or it can be anything for identifying that public key.
- You will get public and private pair. 
- In my `~/.ssh` it created public private pair.
- Used `ssh-copy-id -i <public-key-path> username@hostname` to copy the public file to my linux server. Public key has extension '.pub'.
- I added this in my `~/.ssh/config` to make my login easier: [1]
    Host hostinger
        HostName <hostname>
        User <username>
        Port <portnumber>
        IdentityFile <private-key-path>
- Now I can ssh just by using `ssh <hostname>` and it doesn't ask me for password and is more secure. 
- Also I got to know about remote SSH extension that you can use to connect your VS code interface with your virtual machine.

# Github setup [2]:
- `ssh-keygen -t ed25519 -C <your-email>` to generate a public private key pair.
    - You have to add your private key to your profile under setting in your github dashboard.
-  `eval "$(ssh-agent -s)"` starts an ssh-agent process in the background which will be a service responsible for authorizing our VM when we perform pull, push, clone or other git API calls.
- `ssh-add ~/.ssh/id_ed25519` here we are providing with the keys to be used by ssh-agent.
- `git config --global user.name "<your-name>"` this will setup your commit name.
- `git config --global user.email "<your-email>"` this will setup your commit email.
- `ps aux` or `ps -ejH` or `ps axjf` to see the process tree of running process. Other helpful command for processes and their related metadata: `top` or `htop`.
- `kill <PID>` or `kill -9 <PID>` to kill or forcefully kill a running process.
- **Observation**: Even if you kill ssh-agent process, git will authorize you natively, it doesn't really make use of ssh-agent, so you don't have to worry about starting ssh-agent in each session to work with remote repositories.

# 4. Setting up firewall:
- So the default firewall in ubuntu is ufw. I will be directly using `iptables` because of these reasons:
    - `iptables` is more general `ufw` is ubuntu specific. More specifically it is an interface/abstraction layer on top of iptables.
    - `iptables` is more versatile and granular.
- I wanna see my ufw current rules and different configuration currently applied. Since this is a rented virtual machine there might be some configuration that I would wanna maintain in iptables even if I make the switch.
    - Later realized this is a bad approach as both ufw and iptable-persistent are just interfaces on top of iptable package.
    - You can use `su - <username>` to switch user to root, use: `sudo -i` or `sudo su -`
- `ufw status` to check the ufw's status. `ufw show raw` shows the current rules. There are no default rules and also I saw some docker related rules.
- To get information about a package: `apt info <package-name>` or `apt search <package-name>`.
- **Very important miscalculation**: If the iptables package is not installed on a Linux server, neither ufw nor iptables-persistent will be able to set up or manage firewall rules, as both rely on iptables to function. So conclusively, you can safely remove ufw without worrying about the affect of removal on the state of current rules.
- Now we will install `iptables-persistent`, this is the package that will store the state of iptable. We will be directly defining rule for iptable using the package/keyword `iptable` but instead of ufw, `iptable-persistent` will be storing the state.
- Before performing the below commands it is recommended to have two ssh connection to the same host, in case you lock yourself out.
- Enter the following commands:
```bash
    # Switch to root.
    sudo su -

    # Install iptables-persistent, this will maintain the state of iptables and replace ufw. It automatically removes ufw
    apt install iptables-persistent     

    # Check status of services to confirm everything is going well using:
    systemctl status -l ufw

    # netfilter-persistent is the service. NOTE: service and package name can be different. The package name of netfilter-persistent service is iptables.
    systemctl status -l netfilter-persistent

    # This shows your current iptables firewall rule:
    iptables -L -vn --line-numbers
```
- Now defining the firewall rules. One important thing to note is that the sequence matters. Iptables processes rules in the order they are listed, from top to bottom. When a packet matches a rule, the corresponding action (such as ACCEPT, DROP, or REJECT) is taken, and no further rules are evaluated for that packet.
```bash
    # This is very interesting rule. So this makes your firewall dynamic.
    # -A INPUT: This appends the rule to the INPUT chain, which handles incoming packets.
    # -m state: This uses the state module, which allows matching packets based on their connection state.
    # --state ESTABLISHED: This matches packets that are part of an existing connection. For example, if you initiate a connection from your machine to a web server, the response packets from the web server are considered ESTABLISHED.
    # --state RELATED: This matches packets that are not part of an existing connection but are related to an existing connection. For example, an FTP data transfer connection is related to the initial FTP control connection.
    # -j ACCEPT: This specifies the target action for matching packets, which in this case is to accept them.
    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    
    # The loopback interface is critical for internal communication within the host. Many applications and services rely on the loopback interface to function correctly. Like localhost.
    iptables -A INPUT -p all -i lo -j ACCEPT    # default of -p is all.
    
    # Enables ssh connection over TCP protocol through port 22. If I don't set this, the connection will break when I set the drop policy.
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    
    # Allows DNS resolution via both UDP and TCP.
    iptables -A INPUT -p udp --dport 53 -j ACCEPT
    iptables -A INPUT -p tcp --dport 53 -j ACCEPT

    # Will be changing the default port 22.
    sudo iptables -A INPUT -p tcp --dport <new-ssh-port> -j ACCEPT
    
    # Sets default policy for FORWARD and INPUT CHAIN to DROP.
    iptables -P INPUT DROP
    iptables -P FORWARD DROP
    iptables -nvL --line
    
    # Saves the iptables rules.
    iptables-save > /etc/iptables/rules.v4
    # Alternative. This also prints on the console.
    iptables-save | tee /etc/iptables/rules.v4
    exit
```
- Why did we install iptables-persistent, we didn't use it anywhere. [4]
    - So you have to load the iptables rule manually using this on every reload if you don't use iptables-persistent: `iptables-restore < /etc/iptables.conf`. These are the same rules that you saved using: `iptables-save > /etc/iptables/rules.v4`.
- Try to ssh to your virtual machine and confirm you are not locket out.

# 5. Securing the SSH Connection:
- You have to edit `/etc/ssh/sshd_config` file. Use `sudo vim /etc/ssh/sshd_config`. Or nano or any other editor of your choice.
    - On a side note, in the `/etc/ssh/` directory there are sshd_config and ssh_config files. 
    - `/etc/ssh/sshd_config` is used to configure how SSH connections are made with the device.
    - `/etc/ssh/ssh_config` affects how the device will make an SSH connection to another device.
- In the sshd_config file, look for `# Port 22` and `# PasswordAuthentication yes`. It was on the line 14 and line 18 consecutively in my machine. Uncomment it by removing the '#' and change 22 to whatever port number you wanna use for SSH connection and change 'yes' to 'no' for PasswordAuthentication.
- Once you are done modifying the file use these commands to reload configuration into systemctl memory (reloads dependency tree) [5]: `sudo systemctl daemon-reload` and `sudo systemctl restart ssh`.
- Now delete the port 22 accept rule that we set earlier using `sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT` and save it using `sudo  iptables-save | sudo tee /etc/iptables/rules.v4`.

# Notes: Netfilter (from [3]):
- A firewall works by interacting with the packet filtering hooks in the Linux kernel’s networking stack. These kernel hooks are known as the netfilter framework.
- Every packet that passes through the networking layer (incoming or outgoing) will trigger these hooks, allowing programs to interact with the traffic at key points. The kernel modules associated with iptables register with these hooks in order to ensure that the traffic conforms to the conditions laid out by the firewall rules.
- The following hooks represent these well-defined points in the networking stack:
    - NF_IP_PRE_ROUTING: This hook will be triggered by any incoming traffic very soon after entering the network stack. This hook is processed before any routing decisions have been made regarding where to send the packet.
    - NF_IP_LOCAL_IN: This hook is triggered after an incoming packet has been routed if the packet is destined for the local system.
    - NF_IP_FORWARD: This hook is triggered after an incoming packet has been routed if the packet is to be forwarded to another host.
    - NF_IP_LOCAL_OUT: This hook is triggered by any locally created outbound traffic as soon as it hits the network stack.
    - NF_IP_POST_ROUTING: This hook is triggered by any outgoing or forwarded traffic after routing has taken place and just before being sent out on the wire.

# References:
[1]: https://linuxize.com/post/using-the-ssh-config-file/
[2]: https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent?platform=linux
[3]: https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture
[4]: https://askubuntu.com/questions/66890/how-can-i-make-a-specific-set-of-iptables-rules-permanent
[5]: https://unix.stackexchange.com/questions/364782/what-does-systemctl-daemon-reload-do