# Securing my server:
- I ssh into my server using `ssh root@<ip>` and enter my password when prompted. Now I am logged in as root.
- Here are the things that needs to be done:
    1. I have to update the system.
    2. As a convention according to principle of least privilege you don't stay logged in as root for long and instead use `sudo` to perform root actions. So, I will be creating and admin user.
    3. I wanna setup ssh key pairs to login to my server from my mac without my password. This is more secure and recommended for obvious reasons. Also I have to change some default ssh configurations like port, root login, password login, etc.
    4. I also wanna check my current status of server like the hardware being consumed, processes running, etc.
    5. I also wanna see the current firewall rules and maybe configure them.

# Updating the System