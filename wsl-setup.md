# Setting Up WSL

## WSL Installation (for Windows 11)

Open a new Powershell window as administrator, and type:

```
wsl --install -d Ubuntu
```

Then restart, open a new Ubuntu terminal window, and enter user details. Then type:

```
sudo apt update && sudo apt upgrade
```

## Setting up SSH Server

Install `openssh-server`:

```
sudo apt install openssh-server
```

Set the port for the server by editing `/etc/ssh/sshd_config`:

```
Port 2222  # (you can choose whatever port you want, as long as it is unused)
ListenAddress 0.0.0.0
PasswordAuthentication yes
```

Also, to remove the password requirement when starting the SSH server (useful for scripting), run the following command:

```
sudo visudo
```

Then add the following line to the end of the file:

```
%sudo ALL=NOPASSWD: /usr/sbin/service ssh *
```

## Port Forwarding

By default, any servers running within WSL will not be reachable from outside the virtual machine (including the port for SSH). The following Powershell (`.ps1`) script will start the SSH server and forward the port(s) specified by `$Ports`:

```powershell
# Start SSH Service
wsl sudo service ssh start

# Display all portproxy information
If ($Args[0] -eq "list") {
    netsh interface portproxy show v4tov4;
    exit;
} 

# If elevation needed, start new process
If (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator))
{
  # Relaunch as an elevated process:
  Start-Process powershell.exe "-File",('"{0}"' -f $MyInvocation.MyCommand.Path),"$Args runas" -Verb RunAs
  exit
}

# You should modify '$Ports' for your applications 
$Ports = (2222,80,81)

# Check WSL ip address
wsl hostname -I | Set-Variable -Name "WSL"
$found = $WSL -match '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}';
if (-not $found) {
  echo "WSL2 cannot be found. Terminate script.";
  exit;
}

# Remove and Create NetFireWallRule
Remove-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock';
if ($Args[0] -ne "delete") {
  New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Outbound -LocalPort $Ports -Action Allow -Protocol TCP;
  New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Inbound -LocalPort $Ports -Action Allow -Protocol TCP;
}

# Add each port into portproxy
$Addr = "0.0.0.0"
Foreach ($Port in $Ports) {
    iex "netsh interface portproxy delete v4tov4 listenaddress=$Addr listenport=$Port | Out-Null";
    if ($Args[0] -ne "delete") {
        iex "netsh interface portproxy add v4tov4 listenaddress=$Addr listenport=$Port connectaddress=$WSL connectport=$Port | Out-Null";
    }
}

# Display all portproxy information
netsh interface portproxy show v4tov4;

# Give user to chance to see above list when relaunched start
If ($Args[0] -eq "runas" -Or $Args[1] -eq "runas") {
  Write-Host -NoNewLine 'Press any key to close! ';
  $null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown');
}
```

For convenience, you should place the script at `C:\Scripts\wsl-ports.ps1`. If you would like to add additional ports later on, you can simply modify the `$Ports` variable and re-run the script.

To run the script on startup, create the following `.cmd` file in `%appdata%\Microsoft\Windows\Start Menu\Programs\Startup`:

```
PowerShell -Command "Set-ExecutionPolicy Unrestricted"
PowerShell C:\Scripts\wsl-ports.ps1
```

## Mounting External Storage

By default, Windows should mount any available storage devices in WSL at `/mnt/`, but to be extra sure, add the following line to `/etc/fstab` for any drives which you would like to mount:

```
{drive letter}: /mnt/{drive letter} drvfs defaults 0 0
```

## Installing Docker

If you want things to be easy, just install [Docker Desktop](https://www.docker.com/products/docker-desktop/) and make sure it is using the WSL2 backend in settings. Otherwise, you can manually install and run Docker entirely within WSL (this is probably best if you don't care about having a GUI):

```bash
# Install Docker's package dependencies
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official public PGP key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add the stable Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update the apt package list (for the new Docker repo)
sudo apt update -y

# Install the latest version of Docker (Community Edition) and dependencies
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Allow your user to access the Docker CLI without needing root access
sudo usermod -aG docker $USER
```

To test that everything is working, run the following:

```
sudo docker run hello-world
```
