# Logging in remotely to your firewalled PC at work using a reverse SSH tunnel.

You can't log in remotely to a computer at work or school because their firewall won't let you?
If you have an account on a Unix-like PC where you can run `ssh` and access the internet, you *can* log in to it remotely!
Here's what you can do to gain remote access to your firewalled PC, by automatically logging *from work to your home PC* and then using that connection the other way around: to log to your work PC from home.

# Safety first.

You're going to make it possible to log in to your PC remotely from the internet.
Malicious bots will try to do that several times per hour, so you should protect yourself.
To protect yourself a bit, look at your SSH server configuration, that should be `/etc/ssh/sshd_config`.

Disable root login:

    PermitRootLogin no

Disable logging in with passwords, without a proper key pair:

    PasswordAuthentication no
    PermitEmptyPasswords no

Now only non-root users owning a valid key file will be able to log in, but they still may be able to use `sudo` and `su`.

# Set up SSH.

The SSH server daemon (`sshd`) can be autostarted at your PC if you will be often using it.
To make it possible to log in to your home PC from the internet, you have to forward the port 22 from your router to your PC:

1. Set up static local IP on your home PC (no DHCP).
1. Go to your router config page, and set the forwarding of port 22 to the static local IP of your home PC.

# If you have dynamic outer IP, configure a dynamic DNS.

If you have static IP, you may skip this point.
Your work PC needs to know the IP of your home PC, but the latter keeps changing - about once per day!
This can be fixed using a free "dynamic DNS" service, like [freedns.afraid.org](https://freedns.afraid.org/).
You will have to set it up before continuing this guide.

Roughly, on the mentioned web site:

Make an account and log in.
In the "For Members" menu, the "Subdomains" category: create a subdomain, that will be the URL pointing to your computer - if asked for a "type", pick "A".
In the "Dynamic DNS" category, pick the "quick cron example" - notice how it's mostly commentary. The last line is relevant, it's something like:  

    */10 * * * * /usr/bin/wget --no-check-certificate -O - https://freedns.afraid.org/dynamic/update.php?<SUBDOMAIN ID>= >> /tmp/freedns_<SUBDOMAIN NAME>.log 2>&1 &

This line (with your subdomain ID & name) should do the trick, although it may need adjustments depending on the system.
Put the line in your crontab **at home** with `$ crontab -e`.

This crontab line will make your home PC tell the DNS service its IP every 10 minutes.
Of course, ensure cron is running at your home PC.
Then the DNS will remember it, and it will redirect all requests to the URL you chose to your home PC.

# Check if everything seems to work.

If you have `sshd` configured and running, a port forwarded, and either static IP or dynamic DNS, you can now log in *from your home PC to your home PC* via ssh.
This seems banal, but it checks if everything works.
If you disabled password login (as you should - bots will try to crack your password otherwise, and they likely may if it's not a 12+ character long random string like `0ZT*dpxkQ1#@&AnB2p`) generate a key pair with `ssh-keygen`.
Now authorise your public key by adding it to your authorised keys list (assuming that your key is `~/.ssh/id_rsa`) with:

    $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

This will enable anyone with this key pair (`id_rsa` + `id_rsa.pub`) to log in to your account.
Don't give the private key (`~/.ssh/id_rsa`) to anyone!

Now when you have a private key (or didn't disable password login), try to use ssh:

    $ ssh <YOUR STATIC IP>

or:

    $ ssh <YOUR DYNAMIC DNS URL>

that is, for example: `ssh johndoe.mydomain.org`

If you were able to log in, everything seems to work.
You can log in to your PC remotely.

# Last preparations at home.

You're going to make your work PC automatically connect to your home PC, so then you will be able to use this connection to log back into your work PC from home.
This automatic connecting can be used by anyone who has access to your key files at work to log in to your home PC.
This is a serious security risk, but you can easily fix it.

Make a new user account, and make sure he doesn't belong in any groups - his home group should be his username, and no one else should belong to it.
Make sure he can't use `sudo`: log in as him, and try to use it - you should receive a warning: "<USERNAME> is not in the sudoers file.  This incident will be reported."

First - still at home, as the new user - generate a ssh key pair with no password (just press "enter" when asked for a password), and add the public key to the new user's authorised keys list.
Test if you can login as the new user through your static IP without providing any passwords - the key with no password should suffice.

Take BOTH the private and public key with you to work, and remember or write down the username of the account you created and your static IP or URL.
You may want to leave your home PC running for now to be able to test right away if everything works while at work.

# The fun part: set up a reverse ssh tunnel.

The name may sound confusing, but it's easy to make.

Once at work, one more thing: you should run `ssh-agent` automatically when starting the work PC.
This can be done by adding following code to the beginning of your `~/.bash_profile`:

    SSHAGENT=/usr/bin/ssh-agent
    if [ -z "$SSH_AUTH_SOCK" -a -x "$SSHAGENT" ]; then
        eval `$SSHAGENT`
        trap "kill $SSH_AGENT_PID" 0
    fi

Copy the key pair to your `~/.ssh/` folder.

Test if everything works: try to connect to your home PC from work (let's say the username of the user you created is `jdoe` and your URL is `johndoe.mydomain.org`):

    $ ssh-add
    $ ssh jdoe@johndoe.mydomain.org

You shouldn't be asked for a password, and you should log in as the user you created to your home PC.

Create a short script (let's say you'll save it as `~/bin/tunnel.sh`, the username of the user you created is `jdoe` and your URL is `johndoe.mydomain.org`):

    #!/usr/bin/env bash
    # command that establishes a reverse ssh tunnel
    COMMAND="ssh -N -f -R 2210:localhost:22 jdoe@johndoe.mydomain.org"
    # check if the tunnel already exists - if it does, do nothing
    pgrep -f -x "$COMMAND" &>/dev/null
    if [ $? -ne 0 ]; then
        # use the ssh key files
        ssh-add
        # run the tunnel
        $COMMAND &>/dev/null &
    fi

Now, this script should be run automatically by the work PC.
You can do this with `cron`, by adding a line to your crontab with `crontab -e`:

    */10  * * * * /bin/bash /home/myname/bin/tunnel.sh

This will run your script every 10 minutes, if cron is running at your work PC (it should).

Well, the tunnel should work now!

# Using the tunnel.

With the tunnel established, you can now log in *from home to work*, with:

    $ ssh -p 2210 myname@localhost
    
Where `myname` is your username at work, and `2210` is the port where the tunnel is - specified in the `tunnel.sh` script.
Test if the tunnel actually lets you login - log in normally to your home PC from work, and try to run `$ ssh -p 2210 myname@localhost` - if it will log you "back" into your work PC, the tunnel seems to work!

Remember you will need to authenticate somehow when logging in from home - remember your password or setup a key pair.
`sshd` daemon needs to run both at home and at work.

# One last tweak.

All works, but you may notice your tunnel "collapsing" every once in a while.
This may be because SSH servers by default close all inactive connections.
You may change that by putting both at home and at work (because you log in both ways):

* `TCPKeepAlive yes` in `/etc/ssh/sshd_config`, and 
* `ServerAliveInterval 30` in `/etc/ssh/ssh_config`.

It will make your home PC ask the work PC every 30 seconds to keep the tunnel alive.

# Epilogue.

Good advice: **do not edit your ssh settings remotely!**
Same goes to settings of the tunnel - if it works, don't fix it, or you will make it collapse.
You will lock yourself out, which will lead to frustration.
If you really have to, after saving the changes test if you still can log in before logging out, in a second terminal window!

Disclaimer: author of the guide is only a hobbyist Linux user who knows next to nothing about security.
**You** are responsible for your PCs and your data!
If you are concerned about safety, consult someone who is more experienced in that area about securing your reverse ssh tunnel.
