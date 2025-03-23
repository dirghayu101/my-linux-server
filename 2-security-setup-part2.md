## Disabling Root Login:
1. Update and upgrade your system: `sudo apt update && sudo apt upgrade -y`.
2. Disable root password login:
    1. Run to open the ssh configuration file in VIM or any other editor.`vim /etc/ssh/sshd_config`
    2. Towards the end of this file, there will be `PermitRootLogin` property. Set it to: `PermitRootLogin no`.

## Installing fail2ban:
1. What and why fail2ban: Fail2ban is a security tool that helps protect your server from brute force attacks. It works by monitoring log files for suspicious activity and temporarily banning IP addresses that show malicious signs, such as too many failed login attempts.
2. Install fail2ban: `sudo apt install fail2ban`.
3. Copy the fail2ban configuration file: `cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`.
4. To check if fail2ban is running: `systemctl status fail2ban.service`.
5. There are unlimited configuration that can be done here. Like it gives you a very granular control over a lot of things.