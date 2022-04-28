# Packer template for Windows Server 2019 using vSphere-ISO provider

## Note: this code is compatible with Packer v1.6.x or later. 

This repository contains **HashiCorp Packer** templates to deploy **Windows Server 2019** in **VMware vSphere (with vCenter)**, using the **vsphere-iso** builder.

These templates creates the Template (or VM) directly on the vSphere server and install the latest VMware Tools.

# Content: #

* [autounattend.xml](./autounattend.xml) --> Answer file for unattended Windows setup
* [credentials.json](./credentials.json) --> Credential file
* [windows2019.json](./windows2019.json) --> Windows Server 2019 Packer JSON file Base

Scripts:
* [scripts/disable-network-discovery.cmd](./scripts/disable-network-discovery.cmd) --> Script to Disable network discovery
* [scripts/disable-server-manager.ps1](./scripts/disable-server-manager.ps1) --> Script to Disable Server Manager
* [scripts/disable-winrm.ps1](./scripts/disable-winrm.ps1) --> Script to Disable WinRM
* [scripts/enable-rdp.cmd](./scripts/enable-rdp.cmd) --> Script to Enable Remote Desktop
* [scripts/enable-winrm.ps1](./scripts/enable-winrm.ps1) --> Script to Enable WinRM
* [scripts/install-vm-tools.cmd](./scripts/install-vm-tools.cmd) --> Script to Install VMware Tools
* [scripts/set-temp.ps1](./scripts/set-temp.ps1) --> Script to Set Temp Folders

Tested with **VMware ESX 6.7** and **VMware ESX 7.0** | User: Administrator | Password: Your windows admin password

# Requirements: #

* Packer --> https://www.packer.io

# How to use: #

execute "packer build -var-file=credentials.json windows2019.json"

After you doing a template in VSphere you will have to make a VM from this template.
1. Create a VM from the Windows Template which is provided from the Packer and power it on.
2. Once the Virtual Machine is up and running, login with an RDP client and install Cloudbase-Init:
 - Download the Cloudbase-Init installation binaries from https://github.com/cloudbase/cloudbase-init or downloads page --> https://blogs.vmware.com/management/2019/11/cloudbase-init-windows-initialization.html#:~:text=cloudbase/cloudbase%2Dinit%C2%A0-,or%C2%A0downloads%20page.,-Ensure%20you%20are
 Ensure you are using version 0.9.12.dev72 or greater, which includes the OvfService metadata provider.
 1. Run the CloudbaseInitSetup_x64 installation binary and follow the on-screen instructions. Leave default values, and change Username to Administrator and check Run Cloudbase-Init service as LocalSystem.
 2. Click Install and after the installation completes, edit the configuration files.
 3. Navigate to the chosen installation path, under the conf directory, and edit the file cloudbase-init-unattend.conf. Change metadata_services value to OvfService (with fully qualified class name):
 [DEFAULT]
username=Administrator
groups=Administrators
inject_user_password=true
config_drive_raw_hhd=true
config_drive_cdrom=true
config_drive_vfat=true
bsdtar_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\bin\bsdtar.exe
mtools_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\bin\
verbose=true
debug=true
logdir=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\log\
logfile=cloudbase-init-unattend.log
default_log_levels=comtypes=INFO,suds=INFO,iso8601=WARN,requests=WARN
logging_serial_port_settings=
mtu_use_dhcp_config=true
ntp_use_dhcp_config=true
local_scripts_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\LocalScripts\
metadata_services=cloudbaseinit.metadata.services.ovfservice.OvfService
plugins=cloudbaseinit.plugins.common.mtu.MTUPlugin,cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin,cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin
allow_reboot=false
stop_service_on_exit=false
check_latest_version=false
 4. In the same folder, edit the file cloudbase-init.conf. Change metadata_service to OvfService (with fully qualified class name), and also set first_logon_behaviour and plugins.
 [DEFAULT]
username=Administrator
groups=Administrators
inject_user_password=true
first_logon_behaviour=always
config_drive_raw_hhd=true
config_drive_cdrom=true
config_drive_vfat=true
bsdtar_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\bin\bsdtar.exe
mtools_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\bin\
verbose=true
debug=true
logdir=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\log\
logfile=cloudbase-init.log
default_log_levels=comtypes=INFO,suds=INFO,iso8601=WARN,requests=WARN
logging_serial_port_settings=
mtu_use_dhcp_config=true
ntp_use_dhcp_config=true
local_scripts_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\LocalScripts\
metadata_services=cloudbaseinit.metadata.services.ovfservice.OvfService
plugins=cloudbaseinit.plugins.windows.createuser.CreateUserPlugin,cloudbaseinit.plugins.windows.setuserpassword.SetUserPasswordPlugin,cloudbaseinit.plugins.common.sshpublickeys.SetUserSSHPublicKeysPlugin,cloudbaseinit.plugins.common.userdata.UserDataPlugin
 5. Finally, finish the Cloudbase-Init installation using the options below.
 After the Sysprep process completes, the Virtual Machine will shutdown automatically. Alternatively, it is possible to install Cloudbase-Init in silent mode (unattended).
 6. From vCenter, convert the Virtual Machine to Template.
Additionally, before installing Cloudbase-Init, we can install any tool and make other configurations needed.
After these steps, we have a Windows image ready to use in VMware Cloud Assembly, with Cloudbase-Init prepared to customize our deployments. To customize the guest instance user experience, the user can change the properties shown the configuration files. Most of the properties have default suggested values, except:

- username: in first boot after Sysprep, Administrator account is created with blank password. By choosing Administrator username + SetUserPasswordPlugin + remoteAccess password in the blueprint, Cloudbase-Init will change the blank password.
- first_logon_behaviour: always, the user will be prompted to change the password after first logon.
- metadata_services: by listing only OvfService, Cloudbase-Init won’t try the other metadata services that are not supported in vCenter, having cleaner logs (there won’t be logs of Cloudbase-Init iterating over other metadata services and failing to find them).
- plugins: by listing only the plugins with capabilities supported by OvfService, logs will be cleaner. Also, Cloudbase-init will execute the plugins in this order.
- Run Cloudbase-Init service as LocalSystem: some advanced scripts might require to run Cloudbase-Init service with a dedicated administrator user. If this is the case, it must be selected at installation time.