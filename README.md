# tunneler

### Simplifying reverse SSH tunneling

A reverse tunnel enables access to machines behind firewalls (without any port forwarding) by changing
which end of the tunnel initiates the connection. This works because machines behind firewalls are almost always
enabled to make outbound connections. For example, I have a subnet inside my
home LAN, created by a router/firewall. Machines in the subnet can reach
machines in my LAN and outside my home, but machines in my LAN cannot reach
any of the machines in my subnet. However, by installing a reverse tunnel
host in my LAN, I can have any of the machines in my subnet create reverse
tunnels out to that host in my LAN. Once they have done that I can ssh into
my reverse tunnel host and from there ssh to the host in the subnet through
the reverse tunnel (even though my reverse tunnel host cannot ping or connect
to the subnet host. Maybe a diagram will make this clearer:

![reverse-tunnel](https://github.com/MegaMosquito/tunneler/raw/main/reverse-tunnel.png)

When a host like **Computer Y** above needs to connect to **Computer X**, it's a problem because X is likely at an unknown NAT address and the firewall between them is preventing access. One could open a port on the router, and forward it to X, but that's often not desirable. Instead, X can create a "reverse ssh tunnel" to Y, and once it is established, then Y can connect to X ("forwardly") through the reverse tunnel. The reverse tunnel terminates on the Y side at a port (2201 above) attached to its loopback interface (i.e., `localhost` or `127.0.0.1`). A user on Y could then `ssh user@localhost -p 2201`, with the appropriate credentials for that **user** on host X. Note that any port number (preferably in the range 1024 through 49151, inclusive) can be used (2201 is just an example).

I find setting up reverse tunnels to be error prone, so that's why I created this repo (to make it easier for me). I hope it helps you too.

### Configuring a reverse tunnel receiver

For this to work, a reverse tunnel receiver must be at an address reachable
by the remote host that will be initiating the reverse tunnel creation. I use
a very old and not at all powerful Raspberry Pi model 2B as my reverse tunnel
receiver. I have dedicated this machine for only this purpose.

On your reverse tunnel receiver, configure the SSH daemon, and then create an SSH key pair, using:

```
    $ ssh-keygen
```

I'll assume here that you just gave all the default responses to that command
so that it will have created the files in a well know place on your machine
and that they will not be encrypted. If you know what you are doing then
please feel free to do things differently.

Put the generted **public key** contents (which were generated in:
`/home/pi/.ssh/id_rsa.pub`) iniside this file:

```
    /home/pi/.ssh/authorized_keys
```

Having the public key in there enables any remote host to use the generated
**private key** to remotely login to this host.

To test this, copy the generated private key contents (which were generated
in: `/home/pi/.ssh/id_rsa`) to a remote machine and run this command there:

```
    $ ssh -i id_rsa HOST_IP_ADDRESS
```

(where HOST_IP_ADDRESS is the IP address of your reverse tunnel receiver).

If that works, your reverse tunnel receiver host is ready and you can go
ahead and configure any remote host to create a reverse tunnel to that
receiver.

### Configuring a remote host to create a reverse tunnel

On your remote host, you will first need to (as you did above) copy over the
private key file from your reverse tunnel receiver host and test that you
can create a normal forward ssh connection to the reverse tunnel receiver.

Once that works, clone this repo onto the remote host and edit the
`tunneler` bash script to tel it the address, user, and port you want to
use on the reverse tunne receiver host, and also where you have stored the
private key on this remote host. The `tunneler` script has some comments
at the top to clarify this.

After editing the `tunneler` script, I suggest you configure the remote
host to run this script each time the machine boots up. I do this by
editing the `/etc/rc.local` script that is run by `root` at startup.

I add these lines to that file just before the `exit 0` at the bottom
of the file:

```
# Run the tunneler script in the background
if [ -f /home/pi/git/tunneler/tunneler ]
then
  su pi -c '/home/pi/git/tunneler/tunneler' &
fi
```

The rc.local script is run each time the remote machine starts up. With
these additions, it will first check that this script actually exists (and
since I cloned this repo into my `/home/pi/git` directory, it is at the path
shown above) and if so, run it. If you cloned the repo elsewhere, then edit
the code above to check your script location and to execute your script from
where it is.

I like using this simple `tunneler` script because with it, I can always
connect to my remote hosts anytime, without having to otherwise get to
them to configure a reverse tunnel. Instead, the remote host is **always**
trying to ensure the reverse tunnel is connected. I can simply connect to my
reverse tunnel receiver host, and therefore the remote host's tunnel will
always be accessible to me there on a local port.

If the tunnel crashes for any reason, the "tunneler" script will sleep
briefly to enable the system to release any of the old tunnel's resources,
then it tries again to create the reverse tunnel to your reverse tunnel host.
It continues this cycle forever until the host reboots, when the rc.local
script starts up everything again.

### Using the reverse tunnel

To use the reverse tunnel you need to have the credentials for a user on the remote host. Using the diagram above, a user on Y would require a user name and user credentials to ssh into X. Assuming port 2201 from the diagram above, the user on Y could then use:

```
ssh user@localhost:2201
```

(and then provide the password for `user` on X) in order to connect to X as `user`.

Alternatively, to use an ssh key instead of a password, the user on Y would need an authorized private key for the `user` and could instead use: 

```
ssh -i privatekey user@localhost:2201
```

### Author

Written by Glen Darling, December 2022.

