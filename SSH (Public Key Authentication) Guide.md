>[!hint] Redacted :)
# **Create a user for SSH**
1. `useradd <user>`: Create a user
2. `usermod -aG sudo <user>`: Give user sudo permissions

# **Enable Remote SSH  for Root with Password**
- This is temporary!

1. Enter into the shell for the LXC in Proxmox
2. `nano /etc/ssh/sshd_config`: Edit the *sshd* config file
	1. `PermitRootLogin yes`: Allow root user
	2. `PasswordAuthentication yes`: Allow password login
	3. (Optional) 
		1. `AllowUsers <user1> <user2>`: *Add this line* to allow specific user or users
3. Restart the sshd service for the changes to take effect.  
	- `service ssh restart`

>[!warning] Make sure to enable password remote SSH so you can transfer the key. Then remove the password remote permission.
>

# **Setup Public Key Authentication for SSH**
- Source:https://www.baeldung.com/linux/ssh-setup-public-key-auth

1. **Local**: 
	1. `touch key`: Requires a directory + file to place the key 
	2. `ssh-keygen`: Create SSH Key
		1. `-b 4096` (optional): Specify a length in bytes for the key pair that will be generated
	3. `sudo chmod 400 key`: Make it usable
	
	- This creates two files:
	    - `key`: represents the private key
	    - `key.pub`: represents the public key
	
2. **Local**:
	- Copying the Public Key to a Remote Server with `ssh-copy-id`
	1. `ssh-copy-id -i key.pub <user>@<IP-Addr>`
		1. `-i`: specify a key
	2. `ssh <user>@<IP-Addr> -i key`: Use the key
	
3. **Server**:
	1. Set these permissions back in `nano /etc/ssh/sshd_config`
		1. `PermitRootLogin no`
		2. `PasswordAuthentication no`
		3. `PermitEmptyPasswords no`

## Login with SSH Key

>[!hint] `ssh user@192.168.1.1 -i key`

## Copy to laptop
`scp <user>@<ID-Addr>:<File Location> .`