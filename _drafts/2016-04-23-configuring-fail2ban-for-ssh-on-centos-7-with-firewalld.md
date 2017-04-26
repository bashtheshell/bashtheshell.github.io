---
title: "Configuring Fail2ban for SSH on CentOS 7 with firewalld"
categories:
  - guide
tags:
  - CentOS 7
  - fail2ban
  - firewalld
  - ssh
comments: true
---

I’ve developed this tutorial based on the guides from the following locations:
[https://fedoraproject.org/wiki/Fail2ban_with_FirewallD](https://fedoraproject.org/wiki/Fail2ban_with_FirewallD)  
[http://www.fail2ban.org/wiki/index.php/MANUAL_0_8](http://www.fail2ban.org/wiki/index.php/MANUAL_0_8)

NOTE: If some commands in this tutorial do not work for you, then you probably need superuser privilege to run those commands. You can either run them as root or use sudo command. Not all commands I listed were run with `sudo` as I switched to root at some points while developing this guide.

### **Prelude:**

You'll first need the EPEL repository. You can enable it by running sudo yum install epel-release command. Then you can install the Fail2ban package by issuing `sudo yum install fail2ban`. Be sure to have the 'fail2ban-firewalld' package installed too as we'll be using firewalld. This will include the `/etc/fail2ban/jail.d/00-firewalld.conf` file that will override the 'banaction' definition defined in `jail.conf` file describe later in this document. `ipset` package should also be installed as it's also a component of Netfilter like iptables since firewalld uses ebtables and iptables as indicated by `rpm -qR firewalld` command, which listed the package dependencies. However, both iptables and ebtables are inactive, which is expected when firewalld is installed.

This tutorial only focus on protecting SSH server. You can search online for tutorials on protecting other services after gaining better understanding how fail2ban works in this tutorial. For quick start, you’d only need to create a `jail.local` configuration file in `/etc/fail2ban/` directory with the content below as it’s not recommended to modify the `*.conf` files directly in all of `/etc/fail2ban` subdirectories as the reasons mentioned in the beginning of `jail.conf` file.

```text
[sshd]
enabled = true
```

To do the above in a one-liner command:

`echo -e '[sshd]\nenabled' = true | sudo tee /etc/fail2ban/jail.local 1> /dev/null`

We can add more options such as ‘bantime’ definition for the SSHD section in the above file. Before we get to that part, I want to elaborate on how the above file structure works.

As you can see, `[sshd]` is a filter name in the `jail.local` file, which is associated with the configuration file with the same name in `/etc/fail2ban/filter.d/` directory. The filtering expressions listed under `failregex` definition in `/etc/fail2ban/filter.d/sshd.conf` (assuming `/etc/fail2ban/filter.d/ssh.local` doesn’t exist as expected by default) would be used by the fail2ban server, monitoring for flagged strings in the file indicated in the `logpath` definition in `jail.conf` file. You may notice `%(sshd_log)s`. This is a Python “string interpolation”, and fail2ban is actually written in Python. In our environment (CentOS 7), `sshd_log` translates into `%(syslog_authpriv)s` in `/etc/fail2ban/paths-common.conf` file. `syslog_authpriv` then translates into `/var/log/secure` in `/etc/fail2ban/paths-fedora.conf` file.

If you have at least a banned IP address on your system, then you can run the command: `sudo fail2ban-client status sshd`. You’d see an output similar to below:

```bash
Status for the jail: sshd
|- Filter
|  |- Currently failed:    1
|  |- Total failed:    47
|  `- File list:    /var/log/secure
`- Actions
   |- Currently banned:    1
   |- Total banned:    5
   `- Banned IP list:    10.0.1.8
```

As you can see the `File list` above indicated that `/var/log/secure` file is being used.

Here’s the default configuration in `/etc/fail2ban/jail.conf` file for CentOS 7:

```text
[INCLUDES]
before = paths-fedora.conf
 
[DEFAULT]
ignoreip = 127.0.0.1/8
ignorecommand =
bantime  = 600
findtime  = 600
maxretry = 5
backend = auto
usedns = warn
logencoding = auto
enabled = false
filter = %(__name__)s
destemail = root@localhost
sender = root@localhost
mta = sendmail
protocol = tcp
chain = INPUT
port = 0:65535
banaction = iptables-multiport
 
action_ = %(banaction)s[name=%(__name__)s, bantime="%(bantime)s", port="%(port)s", protocol="%(protocol)s", chain="%(chain)s"]
 
action_mw = %(banaction)s[name=%(__name__)s, bantime="%(bantime)s", port="%(port)s", protocol="%(protocol)s", chain="%(chain)s"]
            %(mta)s-whois[name=%(__name__)s, dest="%(destemail)s", protocol="%(protocol)s", chain="%(chain)s"]
 
action_mwl = %(banaction)s[name=%(__name__)s, bantime="%(bantime)s", port="%(port)s", protocol="%(protocol)s", chain="%(chain)s"]
             %(mta)s-whois-lines[name=%(__name__)s, dest="%(destemail)s", logpath=%(logpath)s, chain="%(chain)s"]
 
action_xarf = %(banaction)s[name=%(__name__)s, bantime="%(bantime)s", port="%(port)s", protocol="%(protocol)s", chain="%(chain)s"]
             xarf-login-attack[service=%(__name__)s, sender="%(sender)s", logpath=%(logpath)s, port="%(port)s"]
 
action_cf_mwl = cloudflare[cfuser="%(cfemail)s", cftoken="%(cfapikey)s"]
                %(mta)s-whois-lines[name=%(__name__)s, dest="%(destemail)s", logpath=%(logpath)s, chain="%(chain)s"]
 
action_blocklist_de  = blocklist_de[email="%(sender)s", service=%(filter)s, apikey="%(blocklist_de_apikey)s"]
 
action_badips = badips.py[category="%(name)s", banaction="%(banaction)s"]
 
action = %(action_)s
 
[sshd]
port    = ssh
logpath = %(sshd_log)s
```

You may notice the `before` definition at the top of the above file. The two lines below are needed for fail2ban to work on CentOS and other Red Hat derivatives:

```text
[INCLUDES]
before = paths-fedora.conf
```

By default, fail2ban has a bantime of 600 seconds (10 minutes) for any banned action, meaning no user can reattempt the connect to the server until the time has passed.

There are few options worth considering modifying:

+ `logpath` is the path to the log file which is provided to the filter.
+ If a retry ‘counter’ meet or exceed the number of matches as indicated in `maxretry`, then a ban action will be triggered upon the offending IP address.
+ If no filter match is made within the duration of `findtime` in seconds, then the retry ‘counter’ will reset to zero.
+ `bantime` is the duration in seconds for the offending IP address to be banned for. Negative bantime second will result in “permanent” ban.

As you can see, the `[DEFAULT]` section in the `/etc/fail2ban/jail.conf` file has the default definitions listed that will propagate to the remaining sections in the file (such as `[SSHD]`) if no definition is defined under those sections. `[SSHD]` defintions in `jail.local` file takes precedence over the `[SSHD]` section in `jail.conf` file. In turn, definitions in `[SSHD]` section in `jail.conf` file take precedence over the `[DEFAULT]` in the same file. For example, `enabled` is set to `false` under `[DEFAULT]` in `jail.conf` file. Since `[SSHD]` section doesn’t have `enabled` defined, fail2ban for SSH service is clearly not enabled.

### **Running Fail2ban:**

Now we can move on to starting the fail2ban, and see it in action. Start the service: `sudo systemctl start fail2ban.service`. If you want fail2ban to start persistently after reboot, then run the command, `sudo systemctl enable fail2ban`.

You may see the fail2ban rules being appended to the firewall. You can verify this by running `sudo firewall-cmd ––direct ––get-all-rules`. You’d get something like below:

`ipv4 filter INPUT 0 -p tcp -m multiport --dports ssh -m set --match-set fail2ban-sshd src -j REJECT --reject-with icmp-port-unreachable`

You can also confirm by running the good old-fashioned reliable command that’s still very much alive today despite iptables not running by issuing the `sudo iptables -L` command.

Do not be alarmed if you believe you have iptables installed. It is definitely installed by default, but not loaded or running. You can verify by paging through the command: `systemctl list-units ––type=service ––all`. You’d see that `firewalld.service` is running and `iptables.service` is inactive.

You may wonder why you aren’t seeing the offending IP addresses listed in the above outputs. See the `––match-set fail2ban-sshd`? This suggests that `ipset` possesses the list of offending IP addresses. You can run the `ipset list fail2ban-sshd` command to view the IP addresses there. Here’s the output:

```bash
Name: fail2ban-sshd
Type: hash:ip
Revision: 1
Header: family inet hashsize 1024 maxelem 65536 timeout 600
Size in memory: 16656
References: 1
Members:
10.0.1.7 timeout 573
```

If you are not seeing the IP addresses listed under `Members:` then it’s likely fail2ban hasn’t banned an IP address within the last `bantime` seconds, which is 600 seconds by default. Please check the fail2ban log file, `/var/log/fail2ban.log`, to see if any banned IP addresses are recently listed.

As you can see in the `ipset` output above, we have the offending IP address followed by `timeout` of 573 seconds. The second is actually represented in real time. To observe the change in real time, run the command, `watch -n 1 ipset list fail2ban-sshd` while receiving the offending IP address.

### **Personal Observation:**

Here’s an interesting observation with the default CentOS behavior when an offending IP address tried to log in the SSH server. I was viewing the `/var/log/secure` and `/var/log/fail2ban.log` log files simultaneously in real time. I intentionally didn’t set up a key-based authentication on the SSH server as I was more interested in how fail2ban works with failed passwords.

Here’s the secure log file:

```text
Apr  3 17:09:54 hostname unix_chkpwd[4802]: password check failed for user (username)
Apr  3 17:09:54 hostname sshd[4788]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=10.0.1.8  user=username
Apr  3 17:09:56 hostname sshd[4788]: Failed password for username from 10.0.1.8 port 60235 ssh2
Apr  3 17:10:09 hostname unix_chkpwd[4996]: password check failed for user (username)
Apr  3 17:10:11 hostname sshd[4788]: Failed password for username from 10.0.1.8 port 60235 ssh2
Apr  3 17:10:32 hostname sshd[4788]: Failed password for username from 10.0.1.8 port 60235 ssh2
Apr  3 17:10:32 hostname sshd[4788]: Connection closed by 10.0.1.8 [preauth]
Apr  3 17:10:32 hostname sshd[4788]: PAM 1 more authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=10.0.1.8  user=username
Apr  3 17:11:17 hostname unix_chkpwd[5141]: password check failed for user (username)
Apr  3 17:11:17 hostname sshd[5123]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=10.0.1.8  user=username
Apr  3 17:11:19 hostname sshd[5123]: Failed password for username from 10.0.1.8 port 60236 ssh2
Apr  3 17:11:33 hostname sshd[5123]: Connection closed by 10.0.1.8 [preauth]
```

Here’s the fail2ban.log file:

```text
2016-04-03 17:09:54,029 fail2ban.filter         [28419]: INFO    [sshd] Found 10.0.1.8
2016-04-03 17:09:56,250 fail2ban.filter         [28419]: INFO    [sshd] Found 10.0.1.8
2016-04-03 17:10:11,863 fail2ban.filter         [28419]: INFO    [sshd] Found 10.0.1.8
2016-04-03 17:10:32,047 fail2ban.filter         [28419]: INFO    [sshd] Found 10.0.1.8
2016-04-03 17:11:17,194 fail2ban.filter         [28419]: INFO    [sshd] Found 10.0.1.8
2016-04-03 17:11:18,274 fail2ban.actions        [28419]: NOTICE  [sshd] Ban 10.0.1.8
2016-04-03 17:11:19,947 fail2ban.filter         [28419]: INFO    [sshd] Found 10.0.1.8
2016-04-03 17:21:19,005 fail2ban.actions        [28419]: NOTICE  [sshd] Unban 10.0.1.8
```

I intentionally attempted to log in to the SSH server with incorrect passwords. The first attempt triggered line 1, 2, and 3 in the `secure` log. Oddly enough, line 1 and 2 in the `secure` log corresponds with only the first line in `fail2ban.log`. The third line in secure log corresponds the second line in `fail2ban.log`. Second attempt at line 4 and 5 in secure log goes with line 3 in `fail2ban.log`. Last attempt at line 6 corresponds the fourth line in `fail2ban.log`. After the last attempt, the current connection terminated, and line 7 and 8 was outputted.

Since fail2ban only noticed 4 failed attempts within the last 600 seconds (duration of `findtime`), one more connection to SSH server was permitted as long as PAM still permit it. The fourth attempt occurred at line 9 and 10 in `secure` log, and those lines corresponds the fifth line in `fail2ban.log`. That fifth line now effectively banned the IP address. However, because the connection is still in session, the user have two more shots to log in with the current connection. Again, we see a duplicate entry in the `fail2ban.log` log as caused by each initial attempt per connection at line 11 in `secure` log that corresponds line 7 in `fail2ban.log` after the `Ban` line. In the last line of `fail2ban.log`, fail2ban unbanned the offending address after 600 seconds (ten minutes) as you can see the time difference from line 6 and 8.

Because of this observation I made, I’ve decided to reduce the `maxretry` to 4 by placing it in the `[SSHD]` section in `/etc/fail2ban/jail.local` file and restart the fail2ban server rather than making the modification elsewhere. After making the changes, you can confirm your change in `/var/log/fail2ban.log`. I successfully tested the change from another host.
