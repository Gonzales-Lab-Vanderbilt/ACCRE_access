# Gonzales Lab ACCRE Storage

## Windows Access & Setup Guide

The Gonzales Lab's ACCRE storage has been migrated from the old AuriStor
filesystem to ACCRE's `/data` storage.

The new lab storage is located at:

``` text
/data/gonzales_lab
```

This is a shortcut to the underlying storage location:

``` text
/v5000/data/gonzales_lab
```

**Use `/data/gonzales_lab` for normal access.**

This guide covers:

-   Accessing ACCRE through your browser
-   Connecting to ACCRE from Windows with SSH
-   Setting up passwordless SSH
-   Creating an easy `ssh accre` shortcut
-   Mounting the lab storage as a normal Windows drive
-   Disconnecting and reconnecting the drive
-   Basic troubleshooting

------------------------------------------------------------------------

## 1. Verify Your ACCRE Access

First, make sure your Vanderbilt account has access to ACCRE and the
Gonzales Lab storage.

Open the ACCRE Visualization Portal:

<https://viz.accre.vu>

Log in using your **Vanderbilt VUNet ID and password**.

Once logged in, select:

**Clusters → ACCRE Shell Access**

This opens a Linux terminal in your browser.

Run:

``` bash
ls -lah /data/gonzales_lab
```

If your account has access, you should see the directories stored in the
Gonzales Lab space.

You can navigate into your own directory, if one has been created for
you:

``` bash
cd /data/gonzales_lab/YOUR_FOLDER
```

Then list its contents:

``` bash
ls -lah
```

> **If you receive `Permission denied`:** Your Vanderbilt account may
> not yet be a member of the `gonzales_lab` ACCRE group. Contact the lab
> administrator/PI or ACCRE support before continuing.

------------------------------------------------------------------------

## 2. Connect to ACCRE from Windows

Windows includes an SSH client, so additional SSH software usually is
not necessary.

Open **Windows PowerShell** and run:

``` powershell
ssh YOUR_VUNET_ID@login.accre.vu
```

For example:

``` powershell
ssh woodsdp@login.accre.vu
```

The first time you connect from a computer, you may be asked whether you
trust the server host key. Verify the host/fingerprint according to
current ACCRE guidance, then accept it if appropriate.

Enter your Vanderbilt password when prompted.

> **Note:** Nothing appears on screen while you type an SSH password.
> You will not see dots, asterisks, or other characters. This is normal.

After successfully connecting, your prompt will change from something
resembling:

``` text
PS C:\Users\YOUR_USERNAME>
```

to something resembling:

``` text
[YOUR_USERNAME@gw02 ~]$
```

You are now connected to ACCRE.

Test access to the lab storage:

``` bash
ls -lah /data/gonzales_lab
```

When finished, disconnect:

``` bash
exit
```

------------------------------------------------------------------------

## 3. Set Up Passwordless SSH

Setting up an SSH key makes regular ACCRE access much easier and is
useful for mounting ACCRE storage as a Windows drive.

### 3.1 Check for an Existing SSH Key

In Windows PowerShell:

``` powershell
Get-ChildItem $env:USERPROFILE\.ssh
```

Look for:

``` text
id_ed25519
id_ed25519.pub
```

If these already exist, you may already have an SSH key. **Do not
overwrite an existing private key unless you know it is safe to do so.**

### 3.2 Generate a New SSH Key

If you do not already have one:

``` powershell
ssh-keygen -t ed25519
```

When asked where to save the key, press **Enter** to accept the default:

``` text
C:\Users\YOUR_WINDOWS_USERNAME\.ssh\id_ed25519
```

For the convenient automatic-mount setup described here, leave the
passphrase blank by pressing **Enter** twice.

Verify the files were created:

``` powershell
Get-ChildItem $env:USERPROFILE\.ssh
```

You should now have:

``` text
id_ed25519
id_ed25519.pub
```

> **Security:** `id_ed25519` is your **private SSH key**. Never email
> it, upload it, or share it. `id_ed25519.pub` is the public key and is
> the file installed on ACCRE.

### 3.3 Install Your Public Key on ACCRE

From Windows PowerShell:

``` powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh YOUR_VUNET_ID@login.accre.vu "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Enter your Vanderbilt password when prompted.

Now test:

``` powershell
ssh YOUR_VUNET_ID@login.accre.vu
```

If everything worked, ACCRE should log you in **without asking for your
Vanderbilt password**.

Exit ACCRE:

``` bash
exit
```

------------------------------------------------------------------------

## 4. Create an Easy `ssh accre` Shortcut

Instead of typing the complete hostname every time, configure:

``` powershell
ssh accre
```

In Windows PowerShell:

``` powershell
notepad $env:USERPROFILE\.ssh\config
```

If Notepad asks whether to create a new file, select **Yes**.

Paste:

``` text
Host accre
    HostName login.accre.vu
    User YOUR_VUNET_ID
    IdentityFile C:/Users/YOUR_WINDOWS_USERNAME/.ssh/id_ed25519
    IdentitiesOnly yes
```

Replace `YOUR_VUNET_ID` and `YOUR_WINDOWS_USERNAME` with your own
information.

Save and close Notepad.

### 4.1 Make Sure the File Is Named `config`

Notepad may save the file as `config.txt`.

Check:

``` powershell
Get-ChildItem $env:USERPROFILE\.ssh -Force
```

You want:

``` text
config
```

not:

``` text
config.txt
```

If necessary:

``` powershell
Rename-Item "$env:USERPROFILE\.ssh\config.txt" "config"
```

### 4.2 Verify the Configuration

Run:

``` powershell
ssh -G accre | Select-String "hostname|user|identityfile"
```

The output should contain your information, for example:

``` text
user YOUR_VUNET_ID
hostname login.accre.vu
identityfile C:/Users/YOUR_WINDOWS_USERNAME/.ssh/id_ed25519
```

Now test:

``` powershell
ssh accre
```

You should connect to ACCRE without entering a password.

From now on, terminal access is simply:

``` powershell
ssh accre
```

------------------------------------------------------------------------

## 5. Install Software for Windows Drive Access

To make the new ACCRE `/data` storage behave similarly to a Windows
network drive, mount it over SSH using **SSHFS-Win**.

Two components are required:

1.  **WinFsp**
2.  **SSHFS-Win**

Install WinFsp first.

### WinFsp

Official releases:

<https://github.com/winfsp/winfsp/releases/latest>

Download the current 64-bit `.msi` installer and use the default
installation options.

### SSHFS-Win

Official releases:

<https://github.com/winfsp/sshfs-win/releases/latest>

Download the current 64-bit `.msi` installer and use the default
installation options.

------------------------------------------------------------------------

## 6. Decide What You Want to Mount

The entire Gonzales Lab storage is:

``` text
/data/gonzales_lab
```

An individual directory may be:

``` text
/data/gonzales_lab/YOUR_FOLDER
```

A specific project may be:

``` text
/data/gonzales_lab/YOUR_FOLDER/PROJECT_FOLDER
```

Mounting only the directory you commonly use keeps the Windows drive
cleaner.

------------------------------------------------------------------------

## 7. Mount ACCRE as a Windows Drive

Open **Windows PowerShell**.

Choose an unused drive letter. The examples below use `Z:`.

To mount your directory:

``` powershell
net use Z: "\\sshfs.kr\YOUR_VUNET_ID@accre\data\gonzales_lab\YOUR_FOLDER"
```

For example:

``` powershell
net use Z: "\\sshfs.kr\woodsdp@accre\data\gonzales_lab\woodsdp"
```

If successful, Windows should report:

``` text
The command completed successfully.
```

Open:

**File Explorer → This PC → Z:**

The contents should correspond to:

``` text
/data/gonzales_lab/YOUR_FOLDER
```

on ACCRE.

### Mount a Specific Project Instead

To make a project itself become `Z:\`:

``` powershell
net use Z: "\\sshfs.kr\YOUR_VUNET_ID@accre\data\gonzales_lab\YOUR_FOLDER\PROJECT_FOLDER"
```

For example:

``` powershell
net use Z: "\\sshfs.kr\woodsdp@accre\data\gonzales_lab\woodsdp\FlexNHP"
```

Then `Z:\` corresponds directly to:

``` text
/data/gonzales_lab/woodsdp/FlexNHP
```

------------------------------------------------------------------------

## 8. Verify the Windows Drive

In Windows PowerShell:

``` powershell
Get-ChildItem Z:\
```

You should see the same files that appear on ACCRE.

Once mounted, normal Windows applications can use paths such as:

``` text
Z:\results\analysis.mat
```

For example, MATLAB:

``` matlab
load('Z:\results\analysis.mat')
```

Python, VS Code, File Explorer, and other Windows applications can
similarly access the mounted drive.

------------------------------------------------------------------------

## 9. Disconnect or Remount the Drive

View currently mapped drives:

``` powershell
net use
```

Disconnect `Z:`:

``` powershell
net use Z: /delete
```

Reconnect later:

``` powershell
net use Z: "\\sshfs.kr\YOUR_VUNET_ID@accre\data\gonzales_lab\YOUR_FOLDER"
```

If `Z:` is already occupied, choose another drive letter:

``` powershell
net use Y: "\\sshfs.kr\YOUR_VUNET_ID@accre\data\gonzales_lab\YOUR_FOLDER"
```

------------------------------------------------------------------------

## 10. Access ACCRE Without Mounting a Drive

For quick command-line access:

``` powershell
ssh accre
```

Then:

``` bash
cd /data/gonzales_lab
```

or:

``` bash
cd /data/gonzales_lab/YOUR_FOLDER
```

List files:

``` bash
ls -lah
```

Disconnect:

``` bash
exit
```

------------------------------------------------------------------------

## 11. Browser-Only Access

If you are away from your normal workstation or do not want to install
anything:

<https://viz.accre.vu>

Log in with your Vanderbilt credentials.

For terminal access:

**Clusters → ACCRE Shell Access**

Then:

``` bash
cd /data/gonzales_lab
```

The Visualization Portal can also be used for graphical file access.

------------------------------------------------------------------------

## 12. Troubleshooting

### `ssh` Asks for Your Vanderbilt Password Every Time

Test:

``` powershell
ssh YOUR_VUNET_ID@login.accre.vu
```

If it still requires a password, reinstall your public key:

``` powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh YOUR_VUNET_ID@login.accre.vu "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Then test again.

### `ssh accre` Uses the Wrong Username

Check:

``` powershell
ssh -G accre | Select-String "hostname|user|identityfile"
```

Verify that it reports:

``` text
user YOUR_VUNET_ID
hostname login.accre.vu
```

If it reports a Windows/domain username instead, make sure the SSH
configuration is actually named:

``` text
config
```

and not:

``` text
config.txt
```

Check with:

``` powershell
Get-ChildItem $env:USERPROFILE\.ssh -Force
```

### A Windows Command Says `command not found`

Make sure you are running Windows commands from **Windows PowerShell**,
not from the ACCRE Linux terminal.

A Windows prompt resembles:

``` text
PS C:\Users\YOUR_USERNAME>
```

An ACCRE prompt resembles:

``` text
[YOUR_USERNAME@gw02 ~]$
```

### `/data/gonzales_lab` Gives `Permission denied`

Your account likely needs to be added to the Gonzales Lab ACCRE group.

Contact the lab administrator/PI or ACCRE support rather than attempting
to change the group directory permissions yourself.

### Windows Says the Drive Letter Is Already in Use

Check:

``` powershell
net use
```

If appropriate, remove an old mapping:

``` powershell
net use Z: /delete
```

Then retry.

### Windows Network-Drive Login Rejects Your Vanderbilt Password

Complete the **SSH key + SSH config** setup first and use the key-based
SSHFS mount:

``` powershell
net use Z: "\\sshfs.kr\YOUR_VUNET_ID@accre\data\gonzales_lab\YOUR_FOLDER"
```

------------------------------------------------------------------------

## 13. Performance Notes

The mounted Windows drive is convenient, but it accesses ACCRE over the
network through SSH.

It is well suited for:

-   Browsing directories
-   Opening scripts
-   Editing code
-   Viewing results
-   Copying individual or moderate-sized files
-   Accessing files from MATLAB/Python
-   General day-to-day file management

Do not assume the SSHFS drive will perform like a local SSD or direct
storage access from an ACCRE compute node.

For very large datasets, sustained high-throughput reads/writes, or
computationally intensive analyses, it is generally preferable to run
the analysis directly on ACCRE.

------------------------------------------------------------------------

# Quick Reference

### Lab storage

``` text
/data/gonzales_lab
```

### ACCRE web portal

<https://viz.accre.vu>

### Connect manually

``` powershell
ssh YOUR_VUNET_ID@login.accre.vu
```

### Connect after SSH setup

``` powershell
ssh accre
```

### Go to lab storage

``` bash
cd /data/gonzales_lab
```

### Check lab storage

``` bash
ls -lah /data/gonzales_lab
```

### Check SSH configuration

``` powershell
ssh -G accre | Select-String "hostname|user|identityfile"
```

### Mount your ACCRE directory as `Z:`

``` powershell
net use Z: "\\sshfs.kr\YOUR_VUNET_ID@accre\data\gonzales_lab\YOUR_FOLDER"
```

### View the mounted drive

``` powershell
Get-ChildItem Z:\
```

### Disconnect the drive

``` powershell
net use Z: /delete
```

### Exit ACCRE

``` bash
exit
```

------------------------------------------------------------------------

## Final Setup

Once configured, normal terminal access is:

``` powershell
ssh accre
```

and normal file access is:

``` text
File Explorer → This PC → Z:
```

Both provide access to the Gonzales Lab's new ACCRE `/data` storage.
