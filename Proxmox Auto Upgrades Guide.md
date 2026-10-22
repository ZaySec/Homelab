>[!hint] Redacted :)

>[!tldr] Being up to date with packages greatly increases security, this is how to automate that for `Proxmox`. (Mailing not included)
- First access a `Proxmox shell` from either SSH or in the browser

## Guide
1. `apt install unattended-upgrades` -> Installs a required package for automation
	
2. `dpkg-reconfigure --priority=low unattended-upgrades` -> Activate automatic updates!
	
3. `systemctl status unattended-upgrades` -> Verify that the service is enabled and running
```shell
root@ZaySec:~# systemctl status unattended-upgrades
● unattended-upgrades.service - Unattended Upgrades Shutdown
     Loaded: loaded (/usr/lib/systemd/system/unattended-upgrades.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-09-14 18:11:20 EDT; 5min ago
 Invocation: b47db6f3fd974cd4b972df5536f9da2f
       Docs: man:unattended-upgrade(8)
   Main PID: 927830 (unattended-upgr)
      Tasks: 1 (limit: 34348)
     Memory: 9.2M (peak: 9.9M)
        CPU: 69ms
     CGroup: /system.slice/unattended-upgrades.service
             └─927830 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
```
	
4. Inside `/etc/apt/apt.conf.d` (`cd <directory>`)
	- `nano 20auto-upgrades` -> Config file for auto updates (Changes optional)
		
	1. Next, we will modify `50unattended-upgrades`
		1. `cp 50unattended-upgrades 99unattended-upgrades` -> This creates a backup of the original config `50unattended-upgrades` and since the copy is `99` it will take priority over `50`
		2. `nano 99unattended-upgrades`
			1. `//Unattended-Upgrade::AutoFixInterruptedDpkg "true";` 
				- -> Uncomment for auto fix
			2. `//Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";` 
				- -> Uncomment to remove unused kernel pacakges
			3. `//Unattended-Upgrade::Remove-New-Unused-Dependencies "true";` 
				- -> Uncomment to remove *new* unused dependencies
			4. `//Unattended-Upgrade::Remove-Unused-Dependencies "false";`
				- -> Uncomment and set to `true`, this is the same as `apt-autoremove`
			5. `//Unattended-Upgrade::Automatic-Reboot "false";`
				- -> Uncomment and set `true` for automatic reboot 
			6. `//Unattended-Upgrade::Automatic-Reboot-Time "03:00";`
				- -> Uncomment for a automatic reboot time
				- Optionally you can change what time it reboots
					- Mine is `"03:00"` aka 3am
			7. XXX 
				1. `//Unattended-Upgrade::Automatic-Reboot-WithUsers "true";`
				- I won't uncomment as I do not want to force any users (most likely you) off the machine
5. `systemctl edit apt-daily-upgrade.timer` -> This is a daily timer that will execute a script
```shell
[Timer]
OnCalendar=*-*-* 2:00
RandomizedDelaySec=10m
Persistent=true
```
- `OnCalendar=*-*-* 2:00`
	- We will update at `02:00` since reboot time is `03:00`, this gives the computer a hour to update before rebooting
- `RandomizedDelaySec=10m` -> Lowered this to 10 minutes
	
6. `systemctl daemon-reload` -> Reload the config files!
	
7. `systemctl restart unattended-upgrades` -> Restart the config
	1. `systemctl status unattended-upgrades` -> Verify it is enabled and active again
	
8. `systemctl status apt-daily-upgrade.timer` -> Verify it is enabled and active again
	
9. `unattended-upgrade --dry-run --debug` -> Run the script in order to find bugs pre-production line

---
# Proxmox Helper Script 

1. [PVE Cron LXC Updater](https://community-scripts.org/categories?category=host-management&preview=cron-update-lxcs)
	1. Proxmox Shell: `bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/cron-update-lxcs.sh)"`