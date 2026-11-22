# Commands Used to Build the Lab

The main commands I ran to build each machine, in order. Passwords are never typed into these commands; they are entered at the prompt, and values in `<angle brackets>` are placeholders.

## DC01 (Windows Server 2022): name, IP, and Active Directory
Run in PowerShell as Administrator.

```powershell
# Static IP on the lab network, then rename and restart
$Alias = (Get-NetAdapter | Where-Object Status -eq 'Up' | Select-Object -First 1).Name
New-NetIPAddress -InterfaceAlias $Alias -IPAddress 192.168.10.10 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias $Alias -ServerAddresses 127.0.0.1
Rename-Computer -NewName "DC01" -Restart
```

```powershell
# Install Active Directory and create the lab.local domain (restarts when done)
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns -SafeModeAdministratorPassword (Read-Host "Set a DSRM password" -AsSecureString) -Force
```

## DC01: OUs, groups, and users
```powershell
$pw = Read-Host "Initial password for all lab users" -AsSecureString
Import-Module ActiveDirectory
$root = "DC=lab,DC=local"

foreach ($ou in "Sales","IT","HR","Workstations","Disabled") {
    New-ADOrganizationalUnit -Name $ou -Path $root -ProtectedFromAccidentalDeletion $false
}
foreach ($g in "Sales","IT","HR") {
    New-ADGroup -Name "GG-$g" -GroupScope Global -GroupCategory Security -Path "OU=$g,$root"
}

$users = @(
 @("Maria","Lopez","Sales"),  @("James","Carter","Sales"), @("Aisha","Khan","Sales"),  @("Luis","Rivera","Sales"),
 @("Dana","Whitfield","IT"),  @("Omar","Diaz","IT"),       @("Priya","Shah","IT"),
 @("Grace","Kim","HR"),       @("Tom","Nguyen","HR"),      @("Elena","Cruz","HR"),
 @("Sam","Patel","Sales"),    @("Nina","Brooks","HR")
)
foreach ($u in $users) {
    $first,$last,$dept = $u
    $sam = ($first.Substring(0,1) + $last).ToLower()
    New-ADUser -Name "$first $last" -GivenName $first -Surname $last `
      -SamAccountName $sam -UserPrincipalName "$sam@lab.local" `
      -Path "OU=$dept,$root" -Department $dept `
      -AccountPassword $pw -Enabled $true
    Add-ADGroupMember -Identity "GG-$dept" -Members $sam
}
```

## DC01: password policy, Group Policy, and file share
```powershell
Install-WindowsFeature GPMC | Out-Null
Import-Module ActiveDirectory, GroupPolicy
$root = "DC=lab,DC=local"

# Domain password and lockout policy
Set-ADDefaultDomainPasswordPolicy -Identity "lab.local" -MinPasswordLength 10 -ComplexityEnabled $true -LockoutThreshold 5 -LockoutDuration "00:30:00" -LockoutObservationWindow "00:30:00"

# Block Control Panel for Sales users
New-GPO -Name "Sales - Block Control Panel" | Out-Null
Set-GPRegistryValue -Name "Sales - Block Control Panel" -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" -ValueName "NoControlPanel" -Type DWord -Value 1
New-GPLink -Name "Sales - Block Control Panel" -Target "OU=Sales,$root"

# Disable USB storage on computers in the Workstations OU
New-GPO -Name "Workstations - Disable USB Storage" | Out-Null
Set-GPRegistryValue -Name "Workstations - Disable USB Storage" -Key "HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR" -ValueName "Start" -Type DWord -Value 4
New-GPLink -Name "Workstations - Disable USB Storage" -Target "OU=Workstations,$root"

# Sales file share, restricted to the GG-Sales group
New-Item -Path "C:\Shares\Sales" -ItemType Directory -Force | Out-Null
New-SmbShare -Name "Sales" -Path "C:\Shares\Sales" -FullAccess "LAB\GG-Sales"
icacls "C:\Shares\Sales" /inheritance:r /grant "LAB\GG-Sales:(OI)(CI)M" "BUILTIN\Administrators:(OI)(CI)F" | Out-Null
```

## CLIENT01 (Windows 11 Enterprise evaluation): join the domain
Run in PowerShell as Administrator.

```powershell
$Alias = (Get-NetIPAddress -AddressFamily IPv4 | Where-Object IPAddress -like '169.254*' | Select-Object -First 1).InterfaceAlias
New-NetIPAddress -InterfaceAlias $Alias -IPAddress 192.168.10.20 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias $Alias -ServerAddresses 192.168.10.10
Resolve-DnsName lab.local
Add-Computer -DomainName "lab.local" -Credential (Get-Credential LAB\Administrator) -Restart
```

On DC01, move the computer into the Workstations OU so the USB policy applies:

```powershell
Move-ADObject -Identity (Get-ADComputer CLIENT01).DistinguishedName -TargetPath "OU=Workstations,DC=lab,DC=local"
```

## TICKET01 (Ubuntu Server 22.04): osTicket
```bash
# Lab network address (second adapter)
sudo tee /etc/netplan/99-lab.yaml <<'YAML'
network:
  version: 2
  ethernets:
    enp0s8:
      addresses: [192.168.10.30/24]
YAML
sudo chmod 600 /etc/netplan/99-lab.yaml
sudo netplan apply

# Web server, PHP, and database
sudo apt install -y apache2 mysql-server php php-mysql php-gd php-imap php-intl php-apcu php-xml php-mbstring php-curl php-zip unzip wget
sudo mysql -e "CREATE DATABASE osticket CHARACTER SET utf8mb4;"
sudo mysql -e "CREATE USER 'ostuser'@'localhost' IDENTIFIED BY '<db-password>';"
sudo mysql -e "GRANT ALL ON osticket.* TO 'ostuser'@'localhost'; FLUSH PRIVILEGES;"

# Download and place osTicket 1.18.1
cd /tmp
wget -O osticket.zip https://github.com/osTicket/osTicket/releases/download/v1.18.1/osTicket-v1.18.1.zip
unzip -o osticket.zip -d osticket
sudo mkdir -p /var/www/html/support
sudo cp -r osticket/upload/* /var/www/html/support/
sudo cp /var/www/html/support/include/ost-sampleconfig.php /var/www/html/support/include/ost-config.php
sudo chmod 0666 /var/www/html/support/include/ost-config.php
sudo chown -R www-data:www-data /var/www/html/support
sudo systemctl restart apache2
```

After finishing the web installer, I locked the installation down:

```bash
sudo chmod 0644 /var/www/html/support/include/ost-config.php
sudo rm -rf /var/www/html/support/setup
```
