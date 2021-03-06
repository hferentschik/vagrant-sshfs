# vagrant-sshfs

This is a vagrant plugin that adds synced folder support for mounting
folders from the Vagrant host into the Vagrant guest via
[SSHFS](https://github.com/libfuse/sshfs). It does this by executing
the `SSHFS` client software within the guest, which creates and SSH
connection from the Vagrant guest back to the Vagrant host. 

The benefits of this approach:
- Works on any host platform and hypervisor type
    - Windows, Linux, Mac OS X
    - Virtualbox, Libvirt, Hyper-V, VMWare
- Seamlessly works on remote Vagrant solutions
    - Works with vagrant aws/openstack/etc.. plugins

The drawbacks with this approach:
- Performance is worse than an implementation like NFS
- There must be `sftp-server` software on the Vagrant host 

`sftp-server` is usually provided by SSH server software so it already
exists on Linux/Mac. On windows you only need to install
[openssh](https://cygwin.com/cgi-bin2/package-cat.cgi?file=x86_64%2Fopenssh%2Fopenssh-7.2p1-1&grep=openssh)
via [cygwin](https://cygwin.com/) and you will get `sftp-server`.

## History

The inspiration for this plugin came from [Fabio Kreusch](https://github.com/fabiokr)
and his [code](https://github.com/fabiokr/vagrant-sshfs) for the original 
vagrant-sshfs Vagrant plugin. The goal of this plugin (as opposed to
the old implementation) is to implement SSHFS as a synced folder
plugin just like the other synced folder plugins (NFS/RSYNC/SMB/VirtualBox).

This plugin was developed mainly by copying the code from the NFS synced 
folder plugin from the Vagrant core code and molding it to fit SSHFS.

## Modes of Operation

### Sharing Vagrant Host Directory to Vagrant Guest - 99% of users

This plugin uses SSHFS slave mounts 
(see [link](https://github.com/dustymabe/vagrant-sshfs/issues/11))
to mount a directory from the Vagrant Host into the Vagrant Guest. It
uses the `sftp-server` software that exists on the host and `sshfs`
running in *slave mode* within the guest to create a connection using
the existing authentication over SSH that vagrant sets up for you.

### Sharing Arbitrary Host Directory to Vagrant Guest - 1% of users

This plugin allows you to share a folder from an arbitrary host to the
Vagrant Guest. This would allow you to do a folder mount to some other
host that may have files that you need. To do this the plugin will run
an SSHFS command from the Guest and connect to the arbitrary host that
must have an SSH daemon running. You must provide the `ssh_host`
option in the Vagrantfile to get this to work. You can use ssh key
forwarding or username/password for authentication for this.

See [Options](#options-specific-to-arbitrary-host-mounting) and 
[Appendix A](#appendix-a-using-keys-and-forwarding-ssh-agent) for
more information.

## Getting Started

In order to use this synced folder implementation perform the
following steps:

### Install Plugin

In order to install the plugin simply run the following command:

```
# vagrant plugin install vagrant-sshfs
```

### Add SSHFS Synced Folder in Vagrantfile

Edit your Vagrantfile to specify a folder to mount from the host into
the guest:

```
config.vm.synced_folder "/path/on/host", "/path/on/guest", type: "sshfs"
```

Now you can simply `vagrant up` and your folder should be mounted in
the guest. For more options that you can add see the [Options](#options) 
section.

## Executing the `vagrant sshfs` Command

The Vagrant SSHFS plugin also supports execution of the `vagrant sshfs`
command from the command line. Executing this command with the `--mount`
option will iterate through the Vagrant file and attempt to mount (via 
SSHFS) any folders that aren't already mounted in the Vagrant guest.
Executing with the `--unmount` option will unmount any mounted folders.

```
vagrant sshfs [--mount|--unmount] [vm-name]
```

## Options

The SSHFS synced folder plugin supports a few options that can be
provided in the `Vagrantfile`. The following sections describe the
options in more detail.

### Generic Options

The SSHFS synced folder plugin supports a few options that can be
provided in the `Vagrantfile`. They are described below:

- `disabled`
    - If set to 'true', ignore this folder and don't mount it.
- `ssh_opts_append`
    - Add some options for the ssh connection that will be established.
    - See the ssh man page for more details on possible options.
- `sshfs_opts_append`
    - Add some options for the sshfs fuse mount that will made
    - See the sshfs man page for more details on possible options.

An example snippet from a `Vagrantfile`:

```
config.vm.synced_folder "/path/on/host", "/path/on/guest",
    ssh_opts_append: "-o Compression=yes -o CompressionLevel=5",
    sshfs_opts_append: "-o auto_cache -o cache_timeout=115200",
    disabled: false
```

### Options Specific to Arbitrary Host Mounting

The following options are only to be used when
[sharing an arbitrary host directory](#sharing-arbitrary-host-directory-to-vagrant-guest---1-of-users)
with the guest. They will be ignored otherwise:

- `ssh_host`
    - The host to connect to via SSH. If not provided this will be 
      detected as the Vagrant host that is running the Vagrant guest.
- `ssh_port`
    - The port to use when connecting. Defaults to port 22.
- `ssh_username`
    - The username to use when connecting. If not provided it is
    detected as the current user who is interacting with Vagrant.
- `ssh_password`
    - The password to use when connecting. If not provided and the
      user is not using SSH keys, then the user will be prompted for
      the password. Please use SSH keys and don't use this option!
- `prompt_for_password`
    - The user can force Vagrant to interactively prompt the user for
      a password by setting this to 'true'. Alternatively the user can
      deny Vagrant from ever prompting for the password by setting
      this to 'false'.

An example snippet from a `Vagrantfile`:

```
config.vm.synced_folder "/path/on/host", "/path/on/guest",
    ssh_host: "somehost.com", ssh_username: "fedora",
    ssh_opts_append: "-o Compression=yes -o CompressionLevel=5",
    sshfs_opts_append: "-o auto_cache -o cache_timeout=115200",
    disabled: false
```

## Appendix A: Using Keys and Forwarding SSH Agent

When [sharing an arbitrary host directory](#sharing-arbitrary-host-directory-to-vagrant-guest---1-of-users)
you may want a completely non-interactive experience. You can either
hard code your password in the Vagrantfile or you can use SSH keys.
A few guides for setting up ssh keys and key forwarding are on Github:
- [Key Generation](https://help.github.com/articles/generating-ssh-keys)
- [Key Forwarding](https://developer.github.com/guides/using-ssh-agent-forwarding/)

The idea is that if `key1` is a key that is authorized to log in to the 
Vagrant host ,meaning there is an entry for `key1` in the `~/.ssh/authorized_keys` 
file, then you should be able to do the following to have a
non-interactive experience with SSH keys and agent forwarding:

Modify the Vagrantfile to forward your SSH agent:

```
config.ssh.forward_agent = 'true'
```

Now set up your agent and add your key to the agent:

```
# eval $(ssh-agent)
# ssh-add /path/to/key1
```

And finally bring up your Vagrant guest:

```
# vagrant up
```


## Appendix B: Development

For local development of this plugin here is an example of how to build 
and install this plugin on your local machine:

```
$ rake build
vagrant-sshfs 0.1.0 built to pkg/vagrant-sshfs-0.1.0.gem.
$ mkdir -p /tmp/gems/gems
$ cp pkg/vagrant-sshfs-0.1.0.gem /tmp/gems/gems/                                                                                                          
$ pushd /tmp/gems/
$ gem generate_index
$ popd
$ vagrant plugin install vagrant-sshfs --plugin-source file:///tmp/gems/
```
