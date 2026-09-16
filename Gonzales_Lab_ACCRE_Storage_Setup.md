# Gonzales Lab ACCRE Storage Access Guide

This guide documents the **tested Windows setup** for accessing Gonzales
Lab ACCRE storage after the AuriStor migration. It includes browser
access, SSH, passwordless SSH, mounting ACCRE as a persistent Windows
drive, a desktop shortcut, and the troubleshooting steps that proved
useful during setup.

> **Lab storage:** `/data/gonzales_lab`\
> **Underlying ACCRE path:** `/v5000/data/gonzales_lab`\
> **Recommended Windows mount:**
> `\\sshfs.r\YOUR_VUNET_ID@login.accre.vu\data\gonzales_lab`

For normal use, use `/data/gonzales_lab`.

------------------------------------------------------------------------

## 1. Confirm ACCRE access

Open the ACCRE Visualization Portal:

https://viz.accre.vu

Log in with your Vanderbilt VUNet ID and password.

Choose **Clusters → ACCRE Shell Access**, then run:

``` bash
ls -lah /data/gonzales_lab
```

If you have access, you should see the Gonzales Lab directories.

If you receive `Permission denied`, your Vanderbilt account likely needs
access to the `gonzales_lab` ACCRE group. Contact the PI/lab
administrator or ACCRE support.

------------------------------------------------------------------------

## 2. Confirm basic SSH access from Windows

Open **PowerShell as Administrator** and run:

``` powershell
ssh YOUR_VUNET_ID@login.accre.vu
```

Example:

``` powershell
ssh woodsdp@login.accre.vu
```

On the first connection, Windows may ask:

``` text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Enter:

``` text
yes
```

Then enter your Vanderbilt password.

Once connected, verify the lab directory:

``` bash
ls -lah /data/gonzales_lab
```

Exit ACCRE:

``` bash
exit
```

------------------------------------------------------------------------

## 3. Set up passwordless SSH

Passwordless SSH is strongly recommended for machines that will
regularly use ACCRE.

### 3.1 Check for an existing key

``` powershell
Get-ChildItem $env:USERPROFILE\.ssh
```

Look for:

``` text
id_ed25519
id_ed25519.pub
```

If these already exist and are the key you intend to use, skip key
generation.

### 3.2 Generate a key

``` powershell
ssh-keygen -t ed25519
```

Press **Enter** to accept the default location:

``` text
C:\Users\YOUR_WINDOWS_USERNAME\.ssh\id_ed25519
```

For unattended/passwordless SSH on a lab-controlled computer, press
**Enter twice** to leave the key passphrase blank.

Do not share the private key:

``` text
id_ed25519
```

The public key is:

``` text
id_ed25519.pub
```

### 3.3 Install the public key on ACCRE

``` powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh YOUR_VUNET_ID@login.accre.vu "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Example:

``` powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh woodsdp@login.accre.vu "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Enter your Vanderbilt password when prompted.

Test the key explicitly:

``` powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 -o IdentitiesOnly=yes YOUR_VUNET_ID@login.accre.vu
```

It should connect without asking for your password.

Then:

``` bash
exit
```

> **For multiple lab computers:** generate a separate SSH key on each
> computer. Do not copy one private key to every machine. Separate keys
> can be revoked independently.

------------------------------------------------------------------------

## 4. Create the `ssh accre` shortcut

Open the SSH config:

``` powershell
notepad $env:USERPROFILE\.ssh\config
```

Paste:

``` text
Host accre
    HostName login.accre.vu
    User YOUR_VUNET_ID
    IdentityFile C:/Users/YOUR_WINDOWS_USERNAME/.ssh/id_ed25519
    IdentitiesOnly yes
```

Example for ACCRE user `woodsdp` on Windows user `gonza`:

``` text
Host accre
    HostName login.accre.vu
    User woodsdp
    IdentityFile C:/Users/gonza/.ssh/id_ed25519
    IdentitiesOnly yes
```

Save and close Notepad.

### Important: check for `config.txt`

``` powershell
Get-ChildItem $env:USERPROFILE\.ssh -Force
```

If Notepad created `config.txt`, rename it:

``` powershell
Rename-Item "$env:USERPROFILE\.ssh\config.txt" "config"
```

Verify the configuration:

``` powershell
ssh -G accre | Select-String "hostname|user|identityfile"
```

Then test:

``` powershell
ssh accre
```

It should connect without asking for a password.

Exit:

``` bash
exit
```

> `ssh accre` working proves that OpenSSH and the SSH key work. It does
> **not** by itself prove that the Windows SSHFS network-drive provider
> is working.

------------------------------------------------------------------------

## 5. Install WinFsp and SSHFS-Win

The Windows drive mount uses **WinFsp + SSHFS-Win**.

Install in this order:

1.  WinFsp x64: https://github.com/winfsp/winfsp/releases/latest
2.  SSHFS-Win x64: https://github.com/winfsp/sshfs-win/releases/latest
3.  Restart Windows.

After restarting, open PowerShell as Administrator.

Verify WinFsp:

``` powershell
Get-Service WinFsp.Launcher
```

It should be `Running`.

If necessary:

``` powershell
Set-Service WinFsp.Launcher -StartupType Automatic
Start-Service WinFsp.Launcher
```

Verify SSHFS-Win:

``` powershell
Test-Path "C:\Program Files\SSHFS-Win\bin\sshfs.exe"
```

Expected:

``` text
True
```

------------------------------------------------------------------------

## 6. Test SSHFS before mapping the lab directory

This test was useful for diagnosing **System error 67**.

First remove any stale mapping for the desired drive letter:

``` powershell
net use Z: /delete
```

It is harmless if Windows says the mapping does not exist.

Check existing mappings:

``` powershell
net use
```

Now test the SSHFS provider using only the ACCRE host:

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu"
```

Example:

``` powershell
net use Z: "\\sshfs.r\woodsdp@login.accre.vu"
```

If this succeeds, WinFsp/SSHFS-Win is working.

Remove the temporary test:

``` powershell
net use Z: /delete
```

### If the test gives `System error 67`

If:

``` powershell
ssh accre
```

works, but:

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu"
```

returns:

``` text
System error 67 has occurred.
The network name cannot be found.
```

then the problem is **not the ACCRE SSH account or the drive letter**.
It is likely the local WinFsp/SSHFS-Win provider.

Check:

``` powershell
Get-Service WinFsp.Launcher
```

``` powershell
Test-Path "C:\Program Files\SSHFS-Win\bin\sshfs.exe"
```

If necessary, uninstall SSHFS-Win and WinFsp, restart, install **WinFsp
first**, install **SSHFS-Win second**, restart again, and repeat the
simple `\\sshfs.r\...` test.

Changing from `Y:` to `Z:` will not fix Error 67 if the SSHFS provider
itself is failing.

------------------------------------------------------------------------

## 7. Map the Gonzales Lab storage

The entire lab directory is:

``` text
/data/gonzales_lab
```

Because this is an absolute Linux path, use the `.r` SSHFS-Win form and
the **full hostname**:

``` text
\\sshfs.r\YOUR_VUNET_ID@login.accre.vu\data\gonzales_lab
```

### Recommended tested mapping

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu\data\gonzales_lab" /persistent:yes
```

Example:

``` powershell
net use Z: "\\sshfs.r\woodsdp@login.accre.vu\data\gonzales_lab" /persistent:yes
```

If prompted, enter the ACCRE/Vanderbilt username and password.

Verify:

``` powershell
Get-ChildItem Z:\
```

You should see the top-level contents of:

``` text
/data/gonzales_lab
```

Check the mapping:

``` powershell
net use
```

> **Important lesson from testing:** use the full hostname
> `login.accre.vu` in the SSHFS UNC path. The OpenSSH alias `accre` may
> work perfectly with `ssh accre` while the Windows SSHFS mapping
> behaves differently.

------------------------------------------------------------------------

## 8. Make credential storage persistent

`/persistent:yes` tells Windows to remember the **drive mapping**, but
the authentication must also survive logout/restart.

Inspect stored SSHFS credentials:

``` powershell
cmdkey /list | Select-String "sshfs" -Context 2,4
```

For a fully persistent setup, look for an SSHFS credential resembling:

``` text
Target: LegacyGeneric:target=\\sshfs.r\YOUR_VUNET_ID@login.accre.vu\data\gonzales_lab
Type: Generic
User: YOUR_VUNET_ID
Local machine persistence
```

The important line is:

``` text
Local machine persistence
```

### If the credential is not stored persistently

Disconnect:

``` powershell
net use Z: /delete
```

Then use:

**File Explorer → This PC → ... → Map network drive**

Set:

``` text
Drive: Z:
Folder: \\sshfs.r\YOUR_VUNET_ID@login.accre.vu\data\gonzales_lab
```

Check:

``` text
Reconnect at sign-in
Connect using different credentials
```

When Windows asks for the SSHFS credentials, enter the ACCRE/Vanderbilt
username and password and select:

``` text
Remember my credentials
```

Then re-check:

``` powershell
cmdkey /list | Select-String "sshfs" -Context 2,4
```

Do not consider a shared/lab workstation fully configured for unattended
restart recovery until the credential shows **Local machine
persistence** and the reboot test below succeeds.

------------------------------------------------------------------------

## 9. Create an ACCRE Desktop `.cmd` launcher

After the persistent `Z:` mapping is working, create a small `.cmd` file
on the Desktop. This gives users a simple **ACCRE - Gonzales Lab** item
to double-click.

The launcher should open the **mapped `Z:` drive**, not the raw SSHFS
UNC path. During testing, sending the raw network path directly to
Explorer sometimes caused Windows to open Documents/OneDrive instead of
ACCRE.

### Recommended method: Notepad

Open **Notepad** and paste:

``` bat
@echo off
start "" explorer.exe Z:\
exit
```

Choose **File → Save As** and use:

``` text
File name: ACCRE - Gonzales Lab.cmd
Save as type: All Files (*.*)
Location: Desktop
```

Make sure the filename ends in `.cmd`, not `.cmd.txt`.

Double-click **ACCRE - Gonzales Lab.cmd**. File Explorer should open:

``` text
Z:\
```

With the recommended mapping in this guide, that is:

``` text
/data/gonzales_lab
```

### Faster PowerShell method

``` powershell
@'
@echo off
start "" explorer.exe Z:\
exit
'@ | Set-Content "$env:USERPROFILE\Desktop\ACCRE - Gonzales Lab.cmd" -Encoding ASCII
```

Then double-click **ACCRE - Gonzales Lab.cmd** on the Desktop.

> **Important:** the `.cmd` does not establish the SSHFS connection or
> make the mapping persistent. It only opens `Z:\`. Complete the
> persistent mapping and credential-storage steps first. After a reboot,
> `Z:` should reconnect independently and the `.cmd` simply opens it.

## 10. Mandatory restart/power-cycle test

A successful mapping during the setup session is **not enough** to prove
persistence.

Restart the computer normally.

After logging back into the same Windows account, run:

``` powershell
net use
```

Then:

``` powershell
Get-ChildItem Z:\
```

Then double-click:

``` text
ACCRE - Gonzales Lab
```

Finally check:

``` powershell
cmdkey /list | Select-String "sshfs" -Context 2,4
```

The setup passes only if:

``` text
Z: still exists
Z:\ opens successfully
/data/gonzales_lab contents are visible
No password is requested
The desktop shortcut opens Z:\
The SSHFS credential remains locally persistent
```

A complete shutdown/power-on test can also be performed after the
restart test.

------------------------------------------------------------------------

## 11. Common problems and fixes

### `ssh accre` works, but `Z:` is missing

The SSH key/OpenSSH setup is working, but the SSHFS drive mapping did
not reconnect.

Check:

``` powershell
net use
```

Then test the provider:

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu"
```

If that succeeds, remove it and recreate the full persistent mapping:

``` powershell
net use Z: /delete
```

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu\data\gonzales_lab" /persistent:yes
```

Then verify Credential Manager persistence and reboot again.

### `System error 67: The network name cannot be found`

Do not keep changing drive letters.

First confirm:

``` powershell
ssh accre
```

Then:

``` powershell
Get-Service WinFsp.Launcher
```

Then:

``` powershell
Test-Path "C:\Program Files\SSHFS-Win\bin\sshfs.exe"
```

Then test:

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu"
```

If SSH works but the simple SSHFS mapping still returns Error 67,
repair/reinstall WinFsp + SSHFS-Win.

### Desktop shortcut opens Documents/OneDrive instead of ACCRE

Do not make the desktop shortcut responsible for opening the raw SSHFS
UNC path.

First make sure:

``` powershell
Get-ChildItem Z:\
```

works.

Then point the shortcut to:

``` text
Z:\
```

or:

``` text
explorer.exe Z:\
```

### `Z:` says the location cannot be found after reboot

Check:

``` powershell
net use
```

and:

``` powershell
cmdkey /list | Select-String "sshfs" -Context 2,4
```

The drive mapping and its credential are separate pieces.
`/persistent:yes` alone does not guarantee that the authentication
credential was saved for future logons.

### Drive letter is already in use

Check:

``` powershell
net use
```

Remove an old mapping if appropriate:

``` powershell
net use Z: /delete
```

Then remap.

------------------------------------------------------------------------

## 12. Browser-only access

If you do not want to install WinFsp/SSHFS-Win:

https://viz.accre.vu

Use:

**Files → Home Directory**

for browser-based file operations, or:

**Clusters → ACCRE Shell Access**

for a terminal.

From the terminal:

``` bash
cd /data/gonzales_lab
```

------------------------------------------------------------------------

## 13. Tested quick-start checklist

For a Windows computer that should mount the **entire Gonzales Lab
directory**:

``` powershell
ssh YOUR_VUNET_ID@login.accre.vu
```

``` powershell
ssh-keygen -t ed25519
```

``` powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | ssh YOUR_VUNET_ID@login.accre.vu "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Create `%USERPROFILE%\.ssh\config`:

``` text
Host accre
    HostName login.accre.vu
    User YOUR_VUNET_ID
    IdentityFile C:/Users/YOUR_WINDOWS_USERNAME/.ssh/id_ed25519
    IdentitiesOnly yes
```

Test:

``` powershell
ssh accre
```

Install **WinFsp**, then **SSHFS-Win**, then restart.

Test the SSHFS provider:

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu"
```

Remove the test:

``` powershell
net use Z: /delete
```

Create the real persistent mapping:

``` powershell
net use Z: "\\sshfs.r\YOUR_VUNET_ID@login.accre.vu\data\gonzales_lab" /persistent:yes
```

Verify:

``` powershell
Get-ChildItem Z:\
```

Check credential persistence:

``` powershell
cmdkey /list | Select-String "sshfs" -Context 2,4
```

Create the desktop shortcut to:

``` text
Z:\
```

Restart Windows and repeat:

``` powershell
net use
```

``` powershell
Get-ChildItem Z:\
```

If `Z:` survives the restart, opens without a password, shows the lab
files, and the SSHFS credential reports **Local machine persistence**,
the Windows ACCRE setup is complete.
