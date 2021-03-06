# Knife vSphere

[![Gem Version](https://badge.fury.io/rb/knife-vsphere.svg)](https://rubygems.org/gems/knife-vsphere)
[![Build Status](https://travis-ci.org/chef-partners/knife-vsphere.svg?branch=master)](https://travis-ci.org/chef-partners/knife-vsphere)

Please refer to the [CHANGELOG](CHANGELOG) for version history and known issues.

* Documentation: <https://github.com/chef-partners/knife-vsphere/blob/master/README.md>
* Source: <http://github.com/chef-partners/knife-vsphere/tree/master>
* Issues: <https://github.com/chef-partners/knife-vsphere/issues>
* Slack: sign up: https://code.vmware.com/slack/ slack channel: #chef
* Mailing list: <https://discourse.chef.io/>

## Installation

If you're using [ChefDK](https://downloads.chef.io/chef-dk/), simply install the
Gem:

```bash
$ chef gem install knife-vsphere
```

If you're using bundler, simply add Chef and `knife-vsphere` to your `Gemfile`:

```ruby
gem 'chef'
gem 'knife-vsphere
```

This plugin is distributed as a Ruby Gem. To install it, run:

```bash
$ gem install knife-vsphere
```

Depending on your system's configuration, you may need to run this command with root privileges.

## Configuration

For initial development, the plugin targets all communication at a vCenter
instance rather than at specific hosts. Only named user authentication is
currently supported; you can add the credentials to your `knife.rb` file:

```ruby
knife[:vsphere_host] = "vcenter-hostname"
knife[:vsphere_user] = "privileged username" # Domain logins may need to be "user@domain.com"
knife[:vsphere_pass] = "your password"       # or %Q(mypasswordwithfunnycharacters)
knife[:vsphere_dc] = "your-datacenter"
```

The vSphere password can also be stored in a base64 encoded version (to
visually obfuscate it) by prepending 'base64:' to your encoded password. For
example:

```ruby
knife[:vsphere_pass] = "base64:Zm9vYmFyCg=="
```

If you get the following error, you may need to disable SSL certificate
checking:

```
ERROR: OpenSSL::SSL::SSLError: SSL_connect returned=1 errno=0
state=SSLv3 read server certificate B: certificate verify failed
```

```ruby
knife[:vsphere_insecure] = true
```

Credentials can also be specified on the command line for multiple vSphere
servers/data centers

## Description:

This is an Chef Knife plugin to interact with VMware's vSphere. This plugin
currently supports the following:

### Listings:

* VMs
* Folders
* Templates
* Datastores
* VLANs (currently requires distributed vswitch)
* Resource Pools and Clusters
* Customization Specifications
* Hosts in a Pool or Cluster


### VM Operations:

* Power on/off
* Clone (with optional chef bootstrap and run list)
* Delete
* VMDK addition
* Migrate
* Connect/disconnect network


### Clone-specific customization options (for Linux guests):

* Destination folder
* CPU core count
* Memory size
* DNS settings
* Hostname / Domain name
* IP addresses / default gateway
* vlan (currently requires distributed vswitch)
* datastore
* resource pool

Note: For Windows guests we can run sysprep

## Basic Examples:

Here are some basic usage examples to help get you started.

- This clones from a VMware template and bootstraps chef into it. It uses
the generic DHCP options.

```bash
$ knife vsphere vm clone MACHINENAME --template TEMPLATENAME --bootstrap --cips dhcp
```

- This clones a vm from a VMware template bootstraps chef, then uses a [Customization template](https://pubs.vmware.com/vsphere-55/index.jsp?topic=%2Fcom.vmware.vsphere.vm_admin.doc%2FGUID-EB5F090E-723C-4470-B640-50B35D1EC016.html)
called "SPEC" to help bootstrap. Also calls a different SSH user and Password.

```bash
$ knife vsphere vm clone MACHINENAME --template TEMPLATENAME --bootstrap --cips dhcp \
  --cspec SPEC --ssh-user USER --ssh-password PASSWORD
```

Note: add a `-f FOLDERNAME` if you put your `--template` in someplace other then
root folder, and use `--dest-folder FOLDERNAME` if you want your VM created in
`FOLDERNAME` rather than the root.

A full basic example of cloning from a folder, and putting it in the "Datacenter Root"
directory is the following:

```bash
$ knife vsphere vm clone MACHINENAME --template TEMPLATENAME -f LOCATIONOFTEMPLATE \
  --bootstrap --start --cips dhcp --dest-folder /
```

- Listing the available VMware templates

```bash
$ knife vsphere template list
Template Name: ubuntu16-template
$ knife vsphere template list -f FOLDERNAME
Template Name: centos7-template
```

- Deleting a machine.

```bash
$ knife vsphere vm delete MACHINENAME (-P will remove from the chef server)
```

# Subcommands

This plugin provides the following Knife subcommands.  Specific command
options can be found by invoking the subcommand with a `--help` flag

## `knife vsphere vm list`

Enumerates the Virtual Machines registered in the target datacenter. Only name
is currently displayed.

```bash
-r, --recursive    - Recurse down through sub-folders to the specified folder
--only-folders     - Print only folder names. Implies recursive
```

## `knife vsphere vm find`

Search for Virtual Machines matching criteria in the specified pool and
display selected fields

CRITERIA:
```bash
--match-ip IP                match ip
--match-name VMNAME          match name
--match-os OS                match os
--match-tools TOOLSSTATE     match tools state
--powered-off                Show only stopped machines
--powered-on                 Show only started machines
```

FIELDS:
```bash
--alarms                     show alarm status
--cpu                        Show cpu
--esx-disk                   Show esx disks
--full-path                  Show full path
--hostname                   show hostname
--ip                         Show primary ip
--ips                        Show all ips assigned to VM with network (format: VLAN1:IP1,VLAN2:IP2)
--os                         Show os details
--os-disks                   Show os disks
--ram                        Show ram
--snapshots                  Show snapshots
--tools                      show tools status
```

Example:

```bash
$ knife vsphere vm find --snapshots --full-path --pool XXX-YYYY --cpu --ram --esx-disk \
    --os-disk --os --match-name my_machine_1 --alarms --tools --ip --ips \
    --match-ip 123 --match-tools toolsOk
```

## `knife vsphere vm state`

Manage power state of a virtual machine, aka turn it off and on

```bash
-s STATE, --state STATE    - The power state to transition the VM into; one of on|off|suspended|reboot
-w PORT, --wait-port PORT  - Wait for VM to be accessible on a port
-g, --shutdown             - Guest OS shutdown
-r, --recursive            - Recurse down through sub-folders to the specified folder to find the VM
```

## `knife vsphere pool list`

Enumerates the Resource Pools and Clusters registered in the target
datacenter.

## `knife vsphere template list`

Enumerates the VM Templates registered in the target datacenter. Only name is
currently displayed.

```bash
-f FOLDER       - Look inside the designated folder, default is the root folder
```

## `knife vsphere customization list`

Enumerates the customization specifications registered in the target
datacenter. Only name is currently displayed.

## `knife vsphere vm clone`

Clones an existing VM template into a new VM instance, optionally applying an
existing customization specification.  If customization arguments such as
`--chost` and `--cdomain` are specified, or if the customization sepcification
fetched from vSphere is considered, a default customization specification will
be attempted.

* For windows, a sysprep based unattended customization in
workgroup mode will be attempted (host name being the VM name unless otherwise
specified).

* For Linux, a fixed named customization using the vmname as the
host name unless otherwise specified.

This command has many options which which to customize your VM.

### Chef bootstrap options

These options alter the way that your VM will be bootstrapped with Chef after it is created. It is not necessary
to bootstrap the VM, but at the very least `--bootstrap` is required to do so.
```bash
--bootstrap - Bootstrap the VM after cloning. Implies --start
--bootstrap-ipv4 - Force using an IPv4 address when a NIC has both IPv4 and IPv6 addresses.
--bootstrap-msi-url URL - Location of the Chef Client MSI if not default from chef.io
--bootstrap-nic INTEGER - Network interface to use when multiple NICs are defined on a template.
--bootstrap-proxy PROXY_URL - The proxy server for the node being bootstrapped
--bootstrap-vault-file VAULT_FILE - A JSON file with a list of vault(s) and item(s) to be updated
--bootstrap-vault-item VAULT_ITEM - A single vault and item to update as "vault:item"
--bootstrap-vault-json VAULT_JSON - A JSON string with the vault(s) and item(s) to be updated
--bootstrap-version VERSION - The version of Chef to install
--distro DISTRO - Bootstrap a distro using a template
--fqdn SERVER_FQDN - Fully qualified hostname for bootstrapping
--hint HINT_NAME[=HINT_FILE] Specify Ohai Hint to be set on the bootstrap target.  Use multiple --hint options to specify multiple hints.
--identity-file IDENTITY_FILE - SSH identity file used for authentication
--json-attributes - A JSON string to be added to the first run of chef-client
--node-name NAME - The Chef node name for your new node
--no-host-key-verify - Disable host key verification
--node-ssl-verify-mode [peer|none] - Whether or not to verify the SSL cert for all HTTPS requests
--prerelease - Install the pre-release chef gems
--run-list RUN_LIST - Comma separated list of roles/recipes to apply
--secret-file SECRET_FILE - A file containing the secret key to use to encrypt data bag item values
--ssh-password PASSWORD - SSH password
--ssh-port PORT - SSH port
--ssh-user USERNAME - SSH username
--sysprep_timeout TIMEOUT - Wait TIMEOUT seconds for sysprep event before continuing with bootstrap
```

### Customization options

These options are related to the customization of the VM by the vSphere agent. They include hardware settings and networking.
```bash
--ccpu CUST_CPU_COUNT - Number of CPUs
--cdomain CUST_DOMAIN - Domain name for customization
--cgw CUST_GW - CIDR IP of gateway for customization
--chostname CUST_HOSTNAME - Unqualified hostname for customization
--cips CUST_IPS - Comma-delimited list of CIDR IPs for customization, or *dhcp* to configure that interface to use DHCP
--cmacs CUST_MACS - Comma-delimited list of MAC addresses, or *auto* to configure that interface to use automatically generated MAC address
--cplugin CUST_PLUGIN_PATH - Path to plugin that implements KnifeVspherePlugin.customize_clone_spec and/or KnifeVspherePlugin.reconfig_vm
--cplugin-data CUST_PLUGIN_DATA - String of data to pass to the plugin.  Use any format you wish.
--cram CUST_MEMORY_GB - Gigabytes of RAM
--cspec CUST_SPEC - The name of any customization specifications that are defined in vCenter to apply
--ctz CUST_TIMEZONE - Timezone in valid 'Area/Location' format
--cvlan CUST_VLANS - Comma-delimited list of VLAN names for the network adapters to join
--disable-customization - By default customizations will be applied to the customization specification (see below).  Disable these convention with this switch
--random-vmname - Creates a random VMNAME starts with vm-XXXXXXXX
--random-vmname-prefix - Change the VMNAME prefix
```

### VMware options

These options alter the way the VM is created, such as to decide where it is placed.
```bash
--datastore STORE    - The datastore into which to put the cloned VM
--datastorecluster STORE - The datastorecluster into which to put the cloned VM
--dest-folder FOLDER - The folder into which to put the cloned VM
--resource-pool POOL|CLUSTER - The resource pool into which to put the cloned VM. Also accepts a cluster name.
--start - Start the VM after cloning.
--sw-uuid SWITCH_UUIDS - Comma-delimited list of virtual switch UUIDs to attach to the network adapters, or *auto* to automatically assign virtual switch
--template TEMPLATE - The source VM / Template to clone from
--template-file TEMPLATE - Full path to location of template to use
```

### Examples

```bash
$ knife vsphere vm clone NewNode --template UbuntuTemplate --cspec StaticSpec \
    --cips 192.168.0.99/24,192.168.1.99/24 \
    --chostname NODENAME --cdomain NODEDOMAIN
```

The customization specification defaults can be disabled using the
`--disable-customization` switch. If you specify a `--cspec` with this option,
that spec will still be applied.

NOTE: if you are specifying a `--cspec` and the cloning process appears to not
be properly applying the spec as defined on vSphere, consider using the
`--disable-customization` as the conventions described above could be
erroneously interfering with the spec as defined on vSphere.

Customization specifications can also be specified in code using the `--cplugin`
and/or `--cplugin-data` arguments.  See the _plugins_ section for examples.

The `--bootstrap-vault-*` options can be used to send `chef-vault` items to be
updated during the hand-off to `knife bootstrap`.

Example using `--bootstrap-vault-json`:

```bash
$  knife vsphere vm clone NewNode UbuntuTemplate --cspec StaticSpec \
    --cips 192.168.0.99/24,192.168.1.99/24 \
    --chostname NODENAME --cdomain NODEDOMAIN \
    --start true --bootstrap true \
    --bootstrap-vault-json '{"passwords":"default","appvault":"credentials"}'
```

## `knife vsphere vm toolsconfig`

```bash
--empty           - allows clearing string properties
```

Sets properties in tools property. See
"https://www.vmware.com/support/developer/vc-sdk/visdk25pubs/ReferenceGuide/vim.vm.ToolsConfigInfo.html"
for available properties and types.

Examples:

```bash
$ knife vsphere vm toolsconfig myvirtualmachine syncTimeWithHost false
$ knife vsphere vm toolsconfig myvirtualmachine pendingCustomization -e
```

## `knife vsphere vm delete NAME`

Deletes an existing VM, removing it from vSphere inventory and deleting from
disk, optionally deleting it from Chef as well.

```bash
--purge|-P        - Delete the client and node from Chef as well
-N                - Specify the name of the node and client to delete if it differs from NAME (requires -P)
```

## `knife vsphere vm snapshot`

Manages the snapshots for an existing VM, allowing for creation, removal, and
reverting of snapshots.

```bash
--list            - List the current tree of snapshots
--create SNAPSHOT - Create a new snapshot off of the current snapshot
--remove SNAPSHOT - Remove a named snapshot.
--revert SNAPSHOT - Revert to a named snapshot.
--revert-current  - Revert to current snapshot.
--start           - Indicates whether to start the VM after a successful revert
--wait            - Indicates whether to wait for creation/removal to complete
--find            - Find the VM instead of specifying the folder with -F
--dump-memory     - Dump the memory when creating the snapshot (default: false)
--quiesce         - Quiesce the VM before snapshotting (default: false)
```

## `knife vsphere vm cdrom`

```bash
--datastore DATASTORE - Datastore the image is stored in
--iso                 - Path and filename of the ISO
--attach              - Attach the iso immediately
--disconnect          - Disconnect any iso currently attached
--recursive           - Search for the VM recursively
--folder              - Search for the VM in the specified folder
--on_boot BOOL        - Set the Attach On Boot Boolean
```

## `knife vsphere vm disk extend`

```bash
--diskname DISKNAME - The name of the disk that will be extended (use when vm has multiple disks)
```

Note: SIZE is in kilobytes

## `knife vsphere vm disk list`

Lists the disks attached to VMNAME

## `knife vsphere datastore list`

Lists all known datastores with capacity and usage

## `knife vsphere datastore maxfree`

Gets the datastore with the most free space

```bash
--regex           - Pattern to match the datastore name
```

## `knife vsphere datastore file`

Uploads files to a datastore and downloads files from a datastore

```bash
--upload-file       - Upload specified local file to remote
--download-file     - Download specified remote file to local
--remote-file FILE  - Remote file name and path
--local-file FILE   - Local file name and path
```

## `knife vsphere datastorecluster list`

Lists all known datastorecluster with capacity and usage

## `knife vsphere datastorecluster maxfree`

Gets the datastorecluster with the most free space

```bash
--regex           - Pattern to match the datastore name
```

## `knife vsphere vm execute`

Executes a program on the guest. Requires vCenter 5.0 or higher.

Command path must be absolute. For Linux guest operating systems, `/bin/bash` is
used to start the program. For Solaris guest operating systems, `/bin/bash` is
used to start the program if it exists. Otherwise `/bin/sh` is used.

Arguments are optional, and allow for redirection in Linux and Solaris.

```bash
--exec-user USERNAME - The username on the guest to execute as.
--exec-passwd PASSWD - The password for the user executing as.
--exec-dir DIRECTORY - Optional: Working directory to execute in. Will default to $HOME of user.
```

## `knife vsphere vm vmdk`

Adds VMDK to VM.

Optional arguments

```bash
--vmdk-type TYPE - VMDK type, "thick" or "thin", defaults to "thin"
```

## `knife vsphere vm markastemplate`

Will traverse the folder tree looking for the VM by name.  By default the
folder inspected with be the root folder.  `--folder` should be specified if
traversing should be in some other folder than the root.  Once found the VM
will be converted into a template.  This means the VM will become a template
and no longer be available as a Virtual Machine.  The name given to the
template will be the name of VM from which it was created.

## `knife vsphere hosts list --pool`

Lists all hosts in given Pool

## `knife vsphere vm migrate`

Migrate VM to resource pool/datastore/host. Resource pool and datastore are
mandatory.

```bash
--folder FOLDER                   - folder in which to search for VM
--resource-pool POOL        - destination resource pool
--dest-host HOST            - destination host (optional)
--dest-datastore DATASTORE  - destination datastore, accesible to HOST
--priority PRIORITY         - migration priority (optional, default defaultPriority )
```

## `knife vsphere vm net`

Set networking state for VMNAME by connecting/disconnecting network
interfaces. Posible states are up/down.

## `knife vsphere vm wait sysprep`

Wait for vm finishing Sysprep

```bash
--sleep SLEEP      - The time in seconds to wait between queries for CustomizationSucceeded event. Default: 60 seconds
--timeout TIMEOUT  - The timeout in seconds before aborting. Default: 300 seconds
```

## `knife vsphere cpu ratio`

Lists the ratio between assigned virtual CPUs and physical CPUs on all hosts.

Example:

```bash
$ knife vsphere cpu ratio
Output:
### Cluster Cluster1 ###
host1.domain.com: 1.8125
host2.domain.com: 2.40625
host3.domain.com: 1.8125

### Cluster Cluster2 ###
host4.domain.com: 1.8125
host5.domain.com: 2.40625
```

## `knife vsphere vlan list`

Lists all the VLANs in the datacenter

## `knife vsphere vlan create`

Creates a vlan (port group on a distributed virtual switch) with the given
name and VLAN ID. If you have multiple distributed switches then use the
`--switch` option to set the switch

# Developing, or using the latest code

The master version of this code may be ahead of the gem itself. If it's in
master you can generally consider it ready to use. To use master instead of
what's published on Ruby gems:

```bash
$ gem uninstall knife-vsphere
$ git clone git@github.com:chef-partners/knife-vsphere.git # or your fork
$ cd knife-vsphere
$ rake build                                           # Take note of the version
$ gem install pkg/knife-vsphere-1.1.1.gem              # Use the version above
```

If you are doing development, then you can run the plugin out of a checked out
copy of the source:

```bash
$ bundle install # only needs to be done once
$ bundle exec knife vsphere ...
```

# Plugins

`knife-vsphere` supports some plugins, currently only for the clone operation.

Plugins let you write code to further customize the operation you are sending to vCenter.

The basic idea is that plugins expose well known methods to `knife`, which are then run at particular times.
The values returned from your methods are passed directly to vSphere.

Below are examples of the potential implementations that would be saved to an rb file and passed in the `--cplugin`
argument.

### cplugin_example.rb

```ruby
class KnifeVspherePlugin
  def data=(cplugin_data)
    # Parse your cplugin_data from the format of your choosing.
  end

  # optional
  def customize_clone_spec(src_config, clone_spec)
    # Customize the clone spec as you see fit.
     return customized_clone_spec
  end

  # optional
  def reconfig_vm(target_vm)
    # Do anything you want in here with the cloned VM.
  end
end
```

#### cplugin_example.rb for cloning a Windows template to VM

```ruby
require 'rbvmomi'

class KnifeVspherePlugin
  attr_accessor :data # rather than defining a data= method

  def customize_clone_spec(src_config, clone_spec)

    if File.exists? data
      customization_data = JSON.parse(IO.read(data)) # see example below
    else
      abort "Customization plugin data file #{data} not accessible"
    end

    cust_guiUnattended = RbVmomi::VIM.CustomizationGuiUnattended(
      :autoLogon => false,
      :autoLogonCount => 1,
      :password => nil,
      :timeZone => customization_data['timeZone']
    )
    cust_identification = RbVmomi::VIM.CustomizationIdentification(
      :domainAdmin => nil,
      :domainAdminPassword => nil,
      :joinDomain => nil
    )
    cust_name = RbVmomi::VIM.CustomizationFixedName(
      :name => customization_data['host_name']
    )
    cust_user_data = RbVmomi::VIM.CustomizationUserData(
      :computerName => cust_name,
      :fullName => customization_data['fullName'],
      :orgName => customization_data['orgName'],
      :productId => customization_data['windows_key']
    )
    cust_sysprep = RbVmomi::VIM.CustomizationSysprep(
      :guiUnattended => cust_guiUnattended,
      :identification => cust_identification,
      :userData => cust_user_data
    )
    dhcp_ip = RbVmomi::VIM.CustomizationDhcpIpGenerator
    cust_ip = RbVmomi::VIM.CustomizationIPSettings(
      :ip => dhcp_ip
    )
    cust_adapter_mapping = RbVmomi::VIM.CustomizationAdapterMapping(
      :adapter => cust_ip
    )
    cust_adapter_mapping_list = [cust_adapter_mapping]
    global_ip = RbVmomi::VIM.CustomizationGlobalIPSettings
    customization_spec = RbVmomi::VIM.CustomizationSpec(
      :identity => cust_sysprep,
      :globalIPSettings => global_ip,
      :nicSettingMap => cust_adapter_mapping_list
    )
    clone_spec.customization = customization_spec
    puts "New clone_spec object :\n #{YAML::dump(clone_spec)}"
    clone_spec
  end

  def reconfig_vm (target_vm)
    puts "In reconfig_vm method.  No actions implemented.."
  end
end
```

### json data file for the above example

```json
{
  "fullName": "Your Company Inc.",
  "orgName": "Your Company Inc.",
  "windows_key": "xxxxx-xxxxx-xxxxx-xxxxx-xxxxx",
  "host_name": "foo_host",
  "timeZone": "100"
}
```


# Getting help

If the software isn't behaving the way you think, or you're having trouble
doing something, we're happy to help. Try this checklist:

*   Are you running the latest version? `gem list knife-vsphere`. You can
    always upgrade with `gem install knife-vsphere`
*   Try running the same command with `-VV` to add additional logging messages
*   Are there any errors in the vSphere console or logs?
*   Search for known issues at
    https://github.com/chef-partners/knife-vsphere/issues


If you're still having problems, head on over to the
[issues](https://github.com/chef-partners/knife-vsphere/issues) page and
create a new issue. Please include:
*   A description of what you are trying to do, what you are seeing
*   The version number of knife-vsphere and of vSphere itself
*   The exact command you're running and the output (sanitize anything you
    don't want public!)

# License

Authors
- Ezra Pagel <ezra@cpan.org>
- Jesse Campbell <hikeit@gmail.com>
- John Williams <john@37signals.com>
- Ian Delahorne <ian@scmventures.se>
- Bethany Erskine <bethany@paperlesspost.com>
- Adrian Stanila <adrian.stanila@sacx.net>
- Raducu Deaconu <rhadoo_io@yahoo.com>
- Leeor Aharon
- Sean Walberg <sean@ertw.com>

```
Copyright
  Copyright © 2011-2013 Ezra Pagel
  Copyright © 2015-2016 Chef Software, Inc
```

```
VMware vSphere is a trademark of VMware, Inc.
```

![Apache License](https://www.apache.org/img/asf_logo.png)

Licensed under the Apache License, Version 2.0 (the "License"); you may not
use this file except in compliance with the License. You may obtain a copy of
the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
License for the specific language governing permissions and limitations under
the License.

Software changes provided by Nicholas Brisebois at Dell SecureWorks. For more
information on Dell SecureWorks security services please browse to
http://www.secureworks.com

© Dell SecureWorks 2015
