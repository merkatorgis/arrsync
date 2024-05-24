# arrsync

Pirate-proof file transfer with rrsync (restricted rsync).

# Use case

You have a server where you need to copy a set of files to, or from. Maybe once;
maybe repeatedly. Maybe from/to another server; maybe from/to your laptop. You
can login to the server (as root, or as a
[sudoer](https://help.ubuntu.com/community/Sudoers)) through
[SSH](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys),
probably with a passphrase-protected private key. That key (and the passphrase)
should be kept private; it should not hang around in places where we might need
it to do some file transfer. You need a secure way to access the server to
transfer files from or to, and it should support file transfers carried out by
unattended scripts as well.

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

# Prerequisites

## npm

The arrsync tool is distributed through the Node Package Manager
([npm](https://www.npmjs.com/)). Npm is used on both the Server and the Clients.
Install it through `apt`:

```
sudo apt update
sudo apt install npm
```

Alternatively, if you don't want to install npm, clone the
[repo](https://github.com/merkatorgis/arrsync) from GitHub, and run the
`arrsync` script from there, wheras with npm you could just run `npm exec
arrsync`, without having to clone it first.

## (r)rsync

On the Clients, we will use [rsync](https://en.wikipedia.org/wiki/Rsync) (remote
sync) to transfer files through SSH to/from the server. On the Server, we will
restrict what rsync may and may not do, using `rrsync` (_restricted rsync_).
Generally, `rrsync` is included in the `rsync` package, which is normally part
of the Linux distribution (`apt install rsync` if you don't already have it).
Try `which rrsync`; if it reports the path to the rrsync command, you're done;
if it fails silently, then you have to take action.

On Ubuntu (jammy) and recent Debian versions, we found it was just there. On
Debian Buster, the rrsync script was installed, but needed to be chmodded
(`chmod +x /usr/share/doc/rsync/scripts/rrsync`) and linked (`ln -s
/usr/share/doc/rsync/scripts/rrsync /usr/local/bin`).

If `man rrsync` doesn't work, you can find the manual here:
https://download.samba.org/pub/rsync/rrsync.1.

# Setup

## 1. Configure the Server

On the Server, just run:

```
sudo npm exec arrsync@latest
```

And choose the Configure Server menu item.

Sudo is needed since we configure things for a another user (normally even a new
user, that the tool will create for you).

You'll be prompted to provide the needed configuration values. Alternatively,
all variables can be passed as command options. See the [usage](./usage) file
for details, or run arrsync and choose Help menu item.

You'll be asked to provide a list of the (public) **IP addresses of the Client
nodes** that you want to allow Read access, and a second list of addresses for
Write access.

Two **public/private key pairs** are generated (one to Read and one to Write).
You'll need the contents of the _private_ key files when configuring the
Clients, which you can find through `sudo cat <user's
home>/.ssh/$host-$user-$mode` (the concrete command printed when the key is
generated), or by running arrsync (with sudo) and choosing List private key
contents.

Two directories are created on the Server in the configured user's home
directory: the `read` directory, where Clients can download files from, and the
`write` directory, where Clients can upload files to. Access for Clients is
restricted to reading from/writing to those two specific directories, by
configuring [rrsync](https://download.samba.org/pub/rsync/rrsync.1) in the
user's [authorized_keys](https://www.ssh.com/academy/ssh/authorized-keys-file)
file.

That `authorized_keys` file is the place where you can later make changes to the
lists of IP addresses for Read and Write access.

## 2. Configure the Client(s)

On each Client, just run:

```
npm exec arrsync@latest
```

And choose Configure Client.

No sudo needed here, since things get configured for the logged-in user.

You'll be prompted to provide the needed configuration values. Alternatively,
most variables can be passed as command options (see the [usage](./usage) file,
or choose Help.

On configuring the Client, you'll need to paste in the **contents of the
_private_ key files** that were generated on the Server. To list these, run
arrsync on the Server (with sudo) and choose List private key contents.

When a Client only needs Read access, and not Write (or the other way around)
just hit Enter without pasting anything for the appropriate key, to render an
empty key file, which effectively blocks access.

Two `Host` entries, referencing the private key files, get configured in the
[~/.ssh/config](https://www.digitalocean.com/community/tutorials/how-to-configure-custom-connection-options-for-your-ssh-client)
file; `$host-$user-read` and `$host-$user-write`. We'll reference these in the
`rsync` commands for downloading and uploading files.

# Use

Once both Server and Clients are set up, downloading and uploading files is done
with general `rsync` commands. Note that rsync is a fairly smart kind of copy;
it will only transfer the _differences_, i.e. the files that were changed,
added, or deleted since the last time rsync was run with the same pair of source
and destination paths.

## Download

On a Client, run this to download all files to the current directory (the `.` at
the end) from the Server's Read directory:

```
rsync -avz $host-$user-read: .
```

E.g. `rsync -avz host.app.com-arrsync-read: .`

The colon is important, as it indicates that we mean a remote (SSH) path, found
in the `~/.ssh/config` file. Adding a subdirectory after the colon is supported.

The `-a` flag is for "archive mode", including recursivity (traversing
subdirectories), and preservation of file permissions.

The `-v` flag is for verbosity, listing all files that are transferred.

The `-z` flag is for compression of the data before transmitting it.

## Upload

Similarly, to upload all files from the current directory (note the `.`) to the
Server's Write directory:

```
rsync -avz . $host-$user-write:
```

E.g. `rsync -avz . host.app.com-arrsync-write:`

Subdirectories after the colon work here as well. If it's just one single
subdirectory, it will be created if it doesn't exist yet, e.g. `rsync -avz .
host.app.com-arrsync-write:/2024-03-15-0912` will work, but `rsync -avz .
host.app.com-arrsync-write:/2024/03/15/0912` will only work if the remote
`~/write/2024/03/15` already exists.

# Troubleshooting

## Authentication failure

For security reasons, the SSH client (which rsync uses to connect with remote
paths) doesn't communicate much details about authentication failures. To
troubleshoot, you can examine the Server log:

```
grep 'sshd' /var/log/auth.log
```

or:

```
grep '$user' /var/log/auth.log | grep 'sshd'
```

## Arrsync versions

Each time you run `npm exec arrsync@latest`, npm checks whether there's a newer
version, and if so, installs it and use that version.

It's important to configure both the Server and its Clients with the same
version of arrsync. You may choose to target a specific version, e.g. `npm exec
arrsync@0.0.7`.
