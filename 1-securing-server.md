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

# 4. Setting up firewall:

# References:
[1]: https://linuxize.com/post/using-the-ssh-config-file/