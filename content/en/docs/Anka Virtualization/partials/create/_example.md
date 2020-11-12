```shell
sudo anka create --ram-size 8G --cpu-count 4 --disk-size 80G \
--app /Applications/Install\ macOS\ Catalina.app 10.15.6
```

> We recommend naming your initial VM after the version of macOS

> **Be aware of the user you're executing Anka CLI commands as.** If you create VMs as root, they won't be available (when running `anka list`) to other users on the system and vice versa

> By default `anka create` creates a VM Template with the username `anka` and password `admin`. Environment variable are available to change these: `ANKA_DEFAULT_USER` and `ANKA_DEFAULT_PASSWD` (be sure to use sudo -E when issuing the create command)

> Catalina VMs require > 4G of RAM and a > 80G disk size

> The VM creation should take around 30 minutes

> **Anka Develop licenses** default your VM in a stopped state (the only available state)

> **Anka Build licenses** default your VM in a suspended state

> **Catalina and lower** VMs are created with SIP/Kext Consent disabled by default. It's strongly advised to keep these settings for optimal Anka performance. If you need to re-enable SIP/Kext Consent, then use this command `anka modify {vmNameOrUUID} set custom-variable sys.csr-active-config 0`

> **Big Sur and higher** VMs are created with SIP/Kext Consent enabled by default.