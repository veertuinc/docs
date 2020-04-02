> ***NOTE***  
> Perform the following steps on the machine that is intended to be a Node.
> "Nodes" are the host machines you want to run the Anka VMs. You can use any Apple hardware for this.

### Install the Anka CLI

#### Download the latest Anka PKG
```shell
curl -L -o Anka.pkg https://veertu.com/downloads/ankabuild-latest
```
- You can find the various Anka Build packages on the [Anka Build download page](https://veertu.com/download-anka-build/).

#### Install the Anka PKG
```shell
sudo installer -pkg Anka.pkg -tgt /
```

#### Verify the installation
```shell 
anka version
```
- Output should be similar to `Anka Build Basic version 2.1.2 (build 112)`.

#### Activate your Anka license
```
sudo anka license activate <key>
```

For Anka CLI commands and options, see the [Command Reference]({{< relref "docs/Anka CLI/commands.md" >}}).

### Create a VM Template

#### Obtain the Mac OS installer

There are multiple ways to obtain the installer .app file for Mac OSX that we'll detail for you below:

1. If you have a pending upgrade to the next minor or patch version of Mac OS:
    - Within `Preferences -> Software Update -> Advanced`, make sure `Download new updates when available` is checked, but `Install macOS updates` is not. While you're still within `Software Update`, click `Update Now` but do not install the next version (Restart) until after you've created the anka VM or the Install .app under /Applications will be deleted.
    - Or on Catalina from the terminal, run `sudo softwareupdate --fetch-full-installer --full-installer-version 10.15.4` to download the installer.
    - You can also use the App Store to download the installer.
2. On any Mac OS version you can use the [installinstallmacos.py](https://github.com/munki/macadmin-scripts) script (requires python):

    - Download and run the script:  
      ```shell
      curl --fail --silent -L -O https://raw.githubusercontent.com/munki/macadmin-scripts/master/installinstallmacos.py
      sudo chmod +x installinstallmacos.py
      sudo ./installinstallmacos.py --raw
      ```

    - The script downloads an image to the location of the .py script, so further steps are necessary to create an Anka VM:
      ```shell
      mkdir -p /tmp/app
      hdiutil attach "<installinstallmacos.py image path>" -mountpoint /tmp/app
      cp -r "/tmp/app/Applications/Install Mac OS Mojave.app" /Applications/
      hdiutil detach /tmp/app -force
      rm -f "<installinstallmacos.py image path>"
      ```
3. Have your local IT department provide a network volume or download links.

#### Run Anka Create to generate the Template
Assuming you chose to download Mojave:
```shell
anka create --app /Applications/Install\ Mac OS\ Mojave.app/ 10.14.6
```

The VM creation should take around 30 minutes. You can continue on to Step 2 while you wait for this to finish.