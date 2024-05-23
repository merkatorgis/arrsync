# arrsync

Pirate-proof filetransfer with rrsync (restricted rsync).

# Use case

You have a server where you need to copy a set of files to, or from. Maybe once;
maybe repeatedly. Maybe from/to another server; maybe from/to your laptop. You
can login to the server as root through SSH, probably with a
passphrase-protected private key. That key (and the passphrase) should be kept
private; it should not hang around in places where we might need it to do some
file transfer. You need a secure way to access the server to transfer files from
or to, and it should support file transfers carried out by unattended scripts as
well.

# Approach

- On the server, we create a new user, dedicated to file transfers.
- We create two public & private key pairs for that user. The private keys are
  without a passphrase.
  - One for clients to write files to this server;
  - One for clients to read files from this server.
- We configure the SSH settings for that user to:
  - Enforce private key authorisation;
  - Disallow any SSH activity other than transferring files (no shell access, no
    port forwarding, etc.);
  - Disallow clients connecting with the Read key any other action than reading
    files from one specific Read directory;
  - Disallow clients connecting with the Write key any other action than writing
    files to one specific Write directory.
- We make it easy for clients to connect as the remote file transfer user to:
  - Download files from the Read directory;
  - Upload files to the Write directory.

Clients (e.g. other servers) can store the Read and/or Write private keys (in a
private place) to be able to connect to the server as the FileTransfer user.
Since the abilities of these keys are very limited, it's not very risky to have
these lying around in this place that (only) people with (root) shell access can
reach.

# Steps

## 1. Ensure the nodes have npm

The arrsync tool is distributed through the Node Package Manager
([npm](https://www.npmjs.com/)). It's needed on both the Server and the Clients.
Install it through `apt`:

```
apt update
apt install npm
```

## 2. Ensure the Server has rrsync

On the Clients, we will use `rsync` (remote sync) to transfer files through SSH
to/from the server. On the Server, we will restrict what rsync may and may not
do, using `rrsync` (_restricted rsync_). Generally, `rrsync` is included in the
`rsync` package, which is normally part of a server installation (`apt install
rsync` if you don't already have it). Try `which rrsync`; if it reports the path
to the rrsync command, you're done; if it fails silently, then you have to take
action.

On Ubuntu (jammy) and recent Debian versions, we found it was just there. On
Debian Buster, the rrsync script was installed, but needed to be chmodded
(`chmod +x /usr/share/doc/rsync/scripts/rrsync`) and linked (`ln -s
/usr/share/doc/rsync/scripts/rrsync /usr/local/bin`).

If `man rrsync` doesn't work, you can find the manual here:
https://download.samba.org/pub/rsync/rrsync.1.

## 3. Configure the server

On the server, run:

```
sudo npm exec arrsync -s
```

Sudo is needed since we configure things for a another user (that normally has
to be created as well).

You'll be prompted to provide the needed configuration values.

Alternatively, all variables can be passed as command options. See the
[usage](./usage) file for details, or run `npm exec arrsync -H`.

## 3. Switch to the new user

```
su filetransfer
```

## 4. List allowed clients

Find out the (public) IP addresses of the SSH clients that you want to connect
for reading/writing files from/to the server. List the addresses
(comma-separated) in two environment variables, like this (use exactly the names
`read` and `write`):

```
read=127.0.0.1,127.0.0.2
write=127.0.0.3,127.0.0.4
```

## 5. Create keys, configure access

Create the keys (press Enter twice for each key, to create it without a
passphrase), and configure access. Just copy and paste this script as-is:

```
for mode in read write; do
    mkdir -pm 700 ~/$mode ~/.ssh
    key=~/.ssh/$HOSTNAME-$USER-$mode
    # Generate a new public/private key pair.
    ssh-keygen -o -a 100 -t ed25519 -f "$key" -C "$mode $USER@$HOSTNAME"
    # Content of public key file.
    pub=$(cat "$key".pub)
    # First letter of $mode value.
    rw=${mode:0:1}
    # The value (a list of IPs) of the variable that is named as the value of mode.
    from=${!mode}
    from=${from:-127.0.0.1,127.0.0.2}
    # Restrict connections authenticating with this key.
    echo "# Restricted ${mode}ing:" >> ~/.ssh/authorized_keys
    echo "command=\"rrsync -${rw}o ~/$mode\",from=\"$from\",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc,no-x11-forwarding $pub" >> ~/.ssh/authorized_keys
done
chmod 600 ~/.ssh/authorized_keys
```

See https://www.ssh.com/academy/ssh/authorized-keys-file if you want to know
more about the `authorized_keys` configuration file.

## 6. Configure clients

1. On the client (probably another server machine that acts as an SSH client
   connecting to the SSH server we just configured), set a variable for the
   hostname of SSH server we want to transform files from/to, like this (use
   exactly the name `filetransfer`, and make sure it has the same value as the
   `$HOSTNAME` we used in the server script above):

```
filetransfer_host=host.app.com
```

2. Optionally configure the remote user name (if it isn't `filetransfer`) like
   this:

```
filetransfer_user=filetransfer
```

3. Configure the connected user's SSH; just copy and paste this script as-is:

```
filetransfer_user=${filetransfer_user:-filetransfer}
mkdir -pm 700 ~/.ssh
for mode in read write; do
    # Host value.
    host=$filetransfer_host-$filetransfer_user-$mode
    # Key file name.
    key=~/.ssh/$host
    touch $key
    # Configure a Host entry.
    echo -n "
Host $host
    HostName $filetransfer_host
    User $filetransfer_user
    IdentityFile $key
" >> ~/.ssh/config
    chmod 600 $key ~/.ssh/config
done
```

See https://www.ssh.com/academy/ssh/config if you want to know more about the
`config` file.

4. Copy the contents of the _private_ key files from the server to the client
   (see "Key file name" in the script above); its just a few lines of text.

## 7. Transfer files

If you don't already have `rsync` on the client, install it through `apt install
rsync`. Once things are configured, the file transfers are secure, _and_ easy:

### 7.1. Read

To download (to the current directory; that's the `./` at the end) all the files
the server has in the `read` directory:

```
rsync -avz $filetransfer-read: ./
```

Remember that the value of `$filetransfer` is the server's hostname, so you
would type e.g. `rsync -avz host.app.com-read: ./`. The colon after the first
parameter indicates to rsync that we mean a remote path. You can also add a
subdirectory after the colon. The `-a` flag is for "archive mode", including
recursivity (traversing subdirectories), and preservation of file
permissions(!). The `-v` flag is for verbosity, listing the files that are
transferred. The `-z` flag is for compression of the data before transmitting
it.

### 7.2. Write

Similarly, to upload all files in the current directory (`./`) to the server's
`write` directory:

```
rsync -avz ./ $filetransfer-write:
```

E.g. `rsync -avz ./ host.app.com-write:`. Subdirectories after the colon work
here as well. If it's just one single subdirectory, it will be created if it
doesn't exist yet, e.g. `rsync -avz ./ host.app.com-write:/2024-03-15-0912` will
work, but `rsync -avz ./ host.app.com-write:/2024/03/15/0912` will only work if
the remote `~/write/2024/03/15` already exists.
