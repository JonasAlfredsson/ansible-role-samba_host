# ansible-role-samba_host

This Ansible role will install Samba (SMB) and configure most of the necessary
stuff for it to work as a file server. Only tested on Debian 10, but should
work well on any derivative.

The things this role will do:

- Install all packages necessary.
- Raise the allowed maximum number of open files (a Windows limitation).
- Configure Samba + its shares.
- Create Samba users.
- Create "Samba user" to "system user" mappings.
- Give you some [useful tips](#good-to-know) about Samba.

Things this role will **not** do:

- Manage firewall settings.
- Create system users (the Samba users needs to already exist as system users).
- Create the share folders/mounts on the system.


## Acknowledgments and Motive
This script is not an entirely original piece of work. I have had a lot of
inspiration from a couple of other similar projects which exists out there:

- [bertvv][2]
- [mrlesmithjr][3]
- [geerlingguy][4]

They all have their own strong points, but I wanted to create something which
combine (what I think) are the best parts of each of these.



# Installation

This repository does not have any dependencies by itself, so just move into
your `roles/` folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-samba_host.git samba_host
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: all
  name: Install Samba and set up shares
  roles:
    - samba_host
```



# Usage

Since the Samba shares are often unique to each individual host, I usual prefer
to define these individual configurations in their respective
`host_vars/{{ ansible_hostname }}` file. However, if you have multiple
identical machines there should not be any problem to define all of this in one
of the `group_vars/` files.

> Check out the [Ansible Hashes](#ansible-hashes) section for some useful
  information about how Ansible (doesn't) handle "merging" of lists.

The following examples have all the available variables included, and all the
default values written out. So any field which is not marked with `# Required`
may be left out of your configuration if you are fine with the defaults.


## Role Variables

```yaml
global_hide_sensitive_output: true
samba_config_file: "/etc/samba/smb.conf"
samba_username_map_file: "/etc/samba/smbusers"
samba_dfree_folder: "/etc/samba/dfree"
```

These should probably not need to be modified, with the exception of the first
one. That one will hide the output from some of the tasks that would otherwise
print the Samba user's passwords in clear text, but in the case of debugging it
might be necessary to set this one to "false".


## Samba Variables
If you need more information about an individual setting I suggest you search
for it [here][1], since that page has really detailed explanations of everything
Samba related.

The structure here will follow the order of the settings as they appear in the
`templates/smb.conf.j2` file. So if there is any confusion between the "Samba
name" and the "Ansible name" of these settings, it should be easy to find the
direct mapping inside that file.

I have added the values the settings will default to if nothing is defined, as
well as comments in case this leads to anything special. However, to be noted
is that there are additional settings inside the `smb.conf` file that is not
changeable via variables, and that mostly depends on that we either do not
support changing them or it would be very unwise to modify it to something
else.

### Global Settings
```yaml
# Browsing/Identification
samba_netbios_name: "{{ ansible_hostname }}"
samba_workgroup: "WORKGROUP"
samba_server_string: "Samba %v"
samba_realm:
samba_wins_support: false
samba_wins_server: []  # Must be empty if wins_support == true.

# Networking
samba_interfaces: []  # Empty == use all interfaces (loopback is always allowed).

# Logging/Debugging
samba_logging: [ "syslog" ]
samba_log_file: "/var/log/samba/log.%m"  # Only if "file" in samba_logging.
samba_max_log_size: 1000  # Only if "file" in samba_logging.
samba_log_level: []

# Authentication/Security
samba_security: "user"
samba_passdb_backend: "tdbsam"
samba_encrypt_passwords: true
samba_obey_pam_restrictions: false
samba_pam_password_change: false
samba_unix_password_sync: false
samba_server_min_protocol: "SMB3_11"
samba_server_max_protocol:
samba_client_min_protocol: "SMB3_11"
samba_client_max_protocol:
samba_client_ipc_min_protocol: "SMB3_11"
samba_client_ipc_max_protocol:
samba_force_encryption: true
samba_host_allow: [ "192.168.0.0/16", "172.16.0.0/12", "10.0.0.0/8" ] # 127.0.0.0/8 always allowed.
samba_host_deny: [ "0.0.0.0/0" ]

# User/Guest Share Permissions
samba_map_to_guest: "Never"
samba_guest_account: "nobody"
samba_usershare_allow_guests: false  # Read the "Guest Access" section before enabling.
samba_read_only: true
samba_create_mask: 0644
samba_force_create_mode: 000

# Misc
samba_unix_extensions: true
samba_wide_links: false  # Will be disabled if samba_unix_extensions == true.
samba_dfree_command:
samba_veto_files: []  # Will also enable "delete veto files" if length > 0.
```

> There are additional sections with detailed information about
  [user/password security](#samba-and-system-users),
  [Samba protocol version](#protocol-version), [guest accounts](#guest-access),
  [logging](#logging) and the [dfree command](#dfree-command) which all could
  be useful to read through once.

### Shares
There are two special share types that can be enabled or disabled;
[`[homes]` and `[printers]`][1]. Please read up on them and take a closer
look at their configuration in the `templates/smb.conf.j2` file before enabling.

```yaml
# [homes]
samba_load_homes: false
# [printers] + [print$]
samba_load_printers: false
```

Then it is possible to define custom user shares. Some of the available
options here are already defined in the [global](#global-settings) section,
but ([almost][6]) anything that is added here will take precedence over those.

```yaml
samba_shares:
  - name:  # Required
    path:  # Required
    comment:
    browseable: true
    read_only:  # Inverse alias of "writable", has precedence.
    writable: false
    valid_users: []
    public:  # Alias of "guest_ok", has precedence.
    guest_ok: false
    write_list: []
    force_group:
    create_mode: 0744
    force_create_mode: 0000
    directory_mode: 0755
    force_directory_mode: 0000
    follow_symlinks: true
    wide_links: false
    dfree_command:
    vfs_objects:
      - name:
        options:
          - name:
            value:
```

Something to also be aware of is that there are some settings in the
[global](#global-settings) section, like [`samba_usershare_allow_guests`][7]
and [`samba_unix_extensions`][8], which blocks some features from being enabled
in the user shares. So read up on them to understand how they work, but if you
just want to enable guest/anonymous access to a share I suggest you take a look
at [this section](#guest-access) for a quick and easy guide. Then there is also
a guide to better understand the `vfs_objects` [further down](#vfs-objects) as
well.

There are [a lot more][1] options available in Samba, but these were those that
I found to be most useful. I accept pull requests if anything else should be
added.

### Users
```yaml
samba_users:
  - name:  # Required
    password:  # Required
```

When creating Samba users it is important that the username already exist as a
system user if `samba_security: "user"` (which is recommended). This means that
you will need to create the system users beforehand, since this role will not
handle that part. Some very useful information and tips regarding this can be
found in [Samba and System Users](#samba-and-system-users) section further down.


### Username Mapping
```yaml
samba_username_map:
  - to:
    from: []
```

When creating Samba users they all need to correspond to the exact name of a
system user, however, afterwards it is possible to map a completely different
Samba login name to this "original" name. This feature may be useful if the
user wants a really [long name][18] to login with through Samba, or you want
multiple spelling variants to map to the same system user. In the following
example all the "from" names will be mapped to the "system-user" username, with
the password that was given to the "original" (i.e. the "to") user.

**Example:**
```yaml
samba_username_map:
  - to: "system-user"
    from:
      - "System-User"
      - "Very-Long-Nickname"
      - "WeIrD-CapiTALization"
```



# Good to Know

This section has a lot of extra information about some of the behavior of both
Ansible and Samba which could be helpful to know about in advance.


## Ansible Hashes
An important thing to remember is that Ansible will overwrite, and not merge,
hashes/dictionaries (like `samba_shares` or `samba_users`) if there are two
with the same name. You can therefore not have a part of them be defined in the
`group_vars/` and then other parts in the `host_vars/`. If you do not like this
behavior you may look into setting [`hash_behaviour = merge`][14], but be aware
that this is not a very [good solution][15]. Instead you should probably look
into the [`combine`][17] filter or the [`merge_vars`][16] action plugin.


## Samba and System Users
Even though a Samba user needs to exist as a system user, before they can be
created, these two actually do not share the same password. What this means
is that when a user logs in via Samba they may use a different password
altogether from what they use when logging in to the system.

This might cause a bit of confusion, so one thing that can be done to alleviate
this it to set [`samba_unix_password_sync: true`][5], which will make so that
Samba automatically updates the system user's password right after a change is
made to the Samba user's password. This is currently only a one way operation,
since Samba has no way to be notified when the system user's password is
changed on its own.

> NOTE: I found some conflicting information about this ([[1][20]], [[2][23]]),
        but it seems like this is only possible if you use PAM and
        [disable password encryption][24].

However, this is pretty good news, since this enables us to create system users
that exist but cannot actually login to the machine. This means it is possible
for the users to drop files on the server, via the Samba share, but you will
not have to worry about them connecting via SSH and executing commands.

Such a "no-login" user can be created by the following command:

```bash
sudo adduser --no-create-home --disabled-password --disabled-login --gecos "" <username>
```

This user will not have a `$HOME` folder, no password and cannot login to the
system. However, it is still possible to create files and folders with
permissions set to this username/uid, which makes it ideal to use when creating
"empty" Samba users on a system.


## Guest Access
Even if you want to allow completely anonymous (i.e. "guest") access, Samba will
still need to map anyone connecting to a share to a specific system user in the
background. This is done in order to know which access rights this "guest"
should have on the system, and by default this "guest" user is mapped to the
"[nobody][25]" system user which has extremely limited abilities.

However, in order to enable this guest account you will need to choose one of
the [available options][26] for the `map to guest` setting. The most common one
used is the "Bad User" one, which means that if you try to login with a username
that does not exist you will automatically be treated as a guest login which
does not need a password.

Then you will need to change the global `samba_usershare_allow_guests` setting,
which is just used as a failsafe so this is not accidentally enabled, before
finally changing the `guest ok` [share](#shares) setting to "true" in order to
actually allow a guest account to access a share.

> :important: Enabling guest access requires [disabling of encryption][33] for
> this share since Samba requires a password to be able to set up an encrypted
> connection. Read more in the [Encrypted Connection](#encrypted-connection)
> section for a workaround.

All the necessary settings are collected in the example below:

**Example:**

```yaml
samba_usershare_allow_guests: true
samba_force_encryption: false

samba_map_to_guest: "Bad User"
samba_guest_account: "nobody"

samba_shares:
  - name:
    path:
    guest_ok: true
```


## Protocol Version
In the [global](#global-settings) settings I have made so that all the
`server`/`client`/`ipc`-`min protocol` defaults to "SMB3_11", since
this should only be lowered in case you are serving something else than Linux
or Windows 10 computers. A list of all supported protocol versions can be found
in the [`server max protocol`][27] section of the manual, and the options are
exactly the same for the client setting as well.

However, something that I experienced when using [cifs][28] to mount Samba
shares on Linux was that it did not negotiate with the server in regards to
which version of the protocol it should use. Instead it just returned this
error message:

```
mount error(95): Operation not supported
```

This is solved by explicitly specifying the protocol version when mounting
("SMB3_11" in this example):

```
sudo mount -t cifs -o vers=3.11 //192.168.0.1/share /home/user/share
```

Or like this in `/etc/fstab` (this mount will also authenticate as "guest" and
wait for the network before it tries to connect):

```
//192.168.0.1/share /home/user/share cifs vers=3.11,guest,_netdev,noexec 0 0
```


## Encrypted Connection
By default the setting `samba_force_encryption: true` will require all clients
to encrypt their connection to the server. Enabling of this is done by adding
the [not well known][32] [`seal`][31] option along with a samba protocol greater
than 3.0 in the mount command, like this:

```
sudo mount -t cifs -o vers=3.11,seal //192.168.0.1/share /home/user/share
```

In Samba [4.15][30] there is also a possibility to tune what type of encryption
is used, but from what I can read it seems like `cifs-utils` only support
AES-128-CCM at the moment.

However, forcing connections to be encrypted conflicts with the `guest ok`
option, since Samba apparently is [unable to create an encrypted connection][33]
for an unauthenticated user (i.e. `guest`) as this account is missing a password
which mean [no encryption keys can be created][34]. A more secure workaround
for this is to create the user "guest" with password "guest" and explaining that
for anyone that want to connect "anonymously" instead of degrading the security
for all users that want to connect to the same share:

```yaml
samba_usershare_allow_guests: false

samba_users:
  - name: "nobody"
    password: "guest"
samba_username_map:
  - to: "nobody"
    from:
      - "guest"

samba_shares:
  - name:
    path:
    valid_users:
      - "nobody"
```


## Logging
The logging level ranges from 1 to 10, where level 1 provides only a small
amount of information and level 10 provides a plethora of low-level information.
Level 2 will provide useful debugging information without wasting disk space.
In practice, you should avoid using log levels greater than 3 unless you are
programming Samba ([source][19]).

There are actually two places where you can define the logging level:

- [`samba_logging`][21]
- [`samba_log_level`][22]

The first one defines the logging backend, i.e. where the messages are printed,
and it is possible to define multiple "outputs" along with the maximum log level
of the messages that are accepted by it.

**Example:**

```yaml
samba_logging:
  - "syslog"
  - "file@3"
```

In this example all messages produced by Samba will be printed to the syslog,
but then only messages which have an "importance level" that is 3 or lower is
written to a file on disk.

In contrast the [`log level`][22] setting enables you to set the verbosity of
different modules of Samba individually.

**Example:**

```yaml
samba_log_level:
  - "3"
  - "passdb:5"
  - "auth:10"
```

In this example we will set the verbosity of all modules in Samba to 3, and
then change the verbosity to 5 for the "passdb" module, and 10 of the "auth"
module.

However, looking back at the previous `logging` example we see that even though
the verbosity of the "auth" module is at level 10, only messages at level 3
or under will be written to the log file (because of the `@3`). But everything
will be written to the "syslog" since it has no limit.


## VFS Objects
The "[`vfs_objects`][9]" setting for the user shares allows you to add
additional features/functions to them, and to show a concrete example of this I
will enable a "[recycling bin][10]" for one of the shares.

The final result, when pushed out to the `smb.conf` file on the server, will
look like this:

```conf
[share_name]
    path = /share/path
    vfs objects = recycle crossrename
    recycle:repository = .recycle
    recycle:keeptree = yes
    recycle:versions = yes
    recycle:touch = yes
    recycle:exclude = *.tmp, *~, *.bak, *.zip, *.rar
    crossrename:sizelimit = 750
```

In the Ansible configuration it will look like this:

> Everything should be treated as a string, since there is no easy way of
  handling a variation of booleans, strings and integers in the same field in
  Jinja2. Booleans may be given as [yes/no, 1/0 or true/false][13].

```yaml
samba_shares:
  - name: "share_name"
    path: "/share/path"
    vfs_objects:
      - name: "recycle"
        options:
          - name: "repository"
            value: ".recycle"
          - name: "keeptree"
            value: "yes"
          - name: "versions"
            value: "yes"
          - name: "touch"
            value: "yes"
          - name: "exclude"
            value: "*.tmp, *~, *.bak, *.zip, *.rar"
      - name: "crossrename"
        options:
          - name: "sizelimit"
            value: "750"
```

What this configuration will create is a folder called `.recycle/` in the root
of the share, i.e. `/share/path/.recycle/`, which will store any file that
has been "deleted". Inside this folder the "original" folder structure will be
kept, which means that if file `/share/path/folder1/file1.txt` is deleted it
will be moved to `/share/path/.recycle/folder1/file1.txt`. Files with the exact
same path will be stored as "Copy #x of filename", and their timestamps will be
updated when they are moved. Finally there is a list with files which will not
be moved to the recycling bin, but rather directly deleted from the disk.

To permanently delete the files it is as simple to just going into the
`.recycle/` folder and deleting them again.

The [`crossrename`][11] object allows Samba to "delete" files that are on
different physical devices, i.e. it allows Samba to move a file from one hard
disk to the `.recycle/` folder that may reside on another disk. This is usually
only a problem if you have a more exotic setup with symlinks for example:

```
/share/
  ├── .recycle/  -> /mnt/disk1/.recycle/
  ├── Pictures/  -> /mnt/disk2/Pictures/
  └── Documents/ -> /mnt/ssd/Documents/
```

> NOTE: This kind of setup will also need `wide_links: true`.

So this setting can be ignored if you only have one disk, are using RAID, or
are pooling disks with something like [mergerfs][12]. The `sizelimit` (in
megabytes) is to stop Samba from "locking up" for too long, since now a
"delete" will take the same time as transferring the file from one disk to
the other one. Files larger than the limit are deleted from disk immediately.


## Dfree Command
In the "exotic" folder setup with symlinks, seen [above](#vfs-objects), Samba
will show the wrong amount of free space that is left in the share. It will
only show the space that is available on the `/` drive, where the `/share/`
folder exist, and not be aware of any of the other disks. It is therefore
possible to define your own custom command that calculates the total amount
of space left in the share.

Samba is actually expecting a file path to be the value of the `dfree_command`
setting, but I have made so that you enter the actual code for the
[dfree command][29] there instead. Ansible will then be the one responsible for
inserting that code into a file (with correct permissions) when deploying.

**Example:**

```yaml
samba_dfree_folder: "/etc/samba/dfree"
samba_shares:
  - name: "Share1"
    path: "/mnt/share1"
    dfree_command: |
      #!/bin/bash
      df | grep disk2 | grep dev | awk '{print $2" "$4}'
```

The example above will create the file `/etc/samba/dfree/Share1` with the
following content:

```bash
#!/bin/bash
df | grep disk2 | grep dev | awk '{print $2" "$4}'
```

and then insert just its file path in the `smb.conf` file instead.

However, this example is very basic, and the code will only calculate the amount
of free space from "disk2". How this actually work is that Samba simply expects
two values to be returned when the command is invoked:

1. The total disk space in blocks.
2. The number of available blocks.
3. (Optional) The blocksize in bytes (default: 1024)

In the example above the `df` command could return something like this:

```
Filesystem               1K-blocks      Used  Available Use% Mounted on
udev                       3965944         0    3965944   0% /dev
tmpfs                       799296      3188     796108   1% /run
/dev/sda2                114851972  30963768   78010956  29% /
/dev/sdb1                961301832  86966276  854783956  10% /mnt/disk2
/dev/sdc1                961301832  34684436  907065796   4% /mnt/disk1
```

Which is then further distilled to just the following returned string:

```
961301832 854783956
```

From this Samba is able to report back a more accurate representation on how
much space is left in the share. That also means that you can perform quite
complicated calculations inside the script if you want to. You only need to
make sure the output follows the required format.






[1]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html
[2]: https://github.com/bertvv/ansible-role-samba
[3]: https://github.com/mrlesmithjr/ansible-samba
[4]: https://github.com/geerlingguy/ansible-role-samba
[5]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#UNIXPASSWORDSYNC
[6]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#HOSTSALLOW
[7]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#USERSHAREALLOWGUESTS
[8]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#UNIXEXTENSIONS
[9]: https://www.samba.org/samba/docs/old/Samba3-HOWTO/VFS.html
[10]: https://www.samba.org/samba/docs/current/man-html/vfs_recycle.8.html
[11]: https://www.samba.org/samba/docs/current/man-html/vfs_crossrename.8.html
[12]: https://github.com/trapexit/mergerfs
[13]: https://askubuntu.com/a/86680
[14]: https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour
[15]: https://medium.com/uptime-99/3-things-ive-learned-about-ansible-the-hard-way-bae341524a86
[16]: https://pypi.org/project/ansible-merge-vars/
[17]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#combining-hashes-dictionaries
[18]: https://www.samba.org/samba/docs/using_samba/ch09.html#samba2-CHP-9-TABLE-2
[19]: https://www.oreilly.com/openbook/samba/book/ch04_08.html
[20]: https://askubuntu.com/questions/153893/samba-and-user-acount-passwords
[21]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#LOGGING
[22]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#LOGLEVEL
[23]: https://ubuntuforums.org/showthread.php?t=1541129
[24]: https://superuser.com/a/999555
[25]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#GUESTACCOUNT
[26]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#MAPTOGUEST
[27]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#SERVERMAXPROTOCOL
[28]: https://linux.die.net/man/8/mount.cifs
[29]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#DFREECOMMAND
[30]: https://www.fatyas.com/wiki/Samba
[31]: https://lists.samba.org/archive/samba/2017-April/207530.html
[32]: https://manpages.ubuntu.com/manpages/impish/man8/mount.cifs.8.html#seal
[33]: https://serverfault.com/a/874499
[34]: https://techcommunity.microsoft.com/blog/filecab/smb-signing-and-guest-authentication/3846679
