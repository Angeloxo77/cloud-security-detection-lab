# ANSIBLE

The next script sets the WinRM service to let Ansible connect to it.

```Powershell
# Enables the WinRM service and sets up the HTTP listener
Enable-PSRemoting -Force

# Opens port 5985 for all profiles
$firewallParams = @{
    Action      = 'Allow'
    Description = 'Inbound rule for Windows Remote Management via WS-Management. [TCP 5985]'
    Direction   = 'Inbound'
    DisplayName = 'Windows Remote Management (HTTP-In)'
    LocalPort   = 5985
    Profile     = 'Any'
    Protocol    = 'TCP'
}
New-NetFirewallRule @firewallParams

# Allows local user accounts to be used with WinRM
# This can be ignored if using domain accounts
$tokenFilterParams = @{
    Path         = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System'
    Name         = 'LocalAccountTokenFilterPolicy'
    Value        = 1
    PropertyType = 'DWORD'
    Force        = $true
}
New-ItemProperty @tokenFilterParams
```

We can also add a HTTPS listener with a self-signed certificate.

```Powershell
# Create self signed certificate
$certParams = @{
    CertStoreLocation = 'Cert:\LocalMachine\My'
    DnsName           = $env:COMPUTERNAME
    NotAfter          = (Get-Date).AddYears(1)
    Provider          = 'Microsoft Software Key Storage Provider'
    Subject           = "CN=$env:COMPUTERNAME"
}
$cert = New-SelfSignedCertificate @certParams

# Create HTTPS listener
$httpsParams = @{
    Path                  = 'WSMan:\localhost\Listener'
    Address               = '*'
    CertificateThumbprint = $cert.Thumbprint
    Enabled               = $true
    Port                  = 5986
    Transport             = 'HTTPS'
    Force                 = $true
}
New-Item @httpsParams

# Opens port 5986 for all profiles
$firewallParams = @{
    Action      = 'Allow'
    Description = 'Inbound rule for Windows Remote Management via WS-Management. [TCP 5986]'
    Direction   = 'Inbound'
    DisplayName = 'Windows Remote Management (HTTPS-In)'
    LocalPort   = 5986
    Profile     = 'Any'
    Protocol    = 'TCP'
}
New-NetFirewallRule @firewallParams
```
---
## Inventory

As we have two different OS on our assets, we will be accessing differently. Windows requires user and password while in the other hand we will use a SSH key file for Linux.

The playbook for the inventory would look something like this:
´´´Yaml
all:
  hosts:
    wserver01:
      ansible_host: 10.0.0.2
      ansible_user: user
      ansible_password: "{{ vault_win_pass }}"
      ansible_connection: winrm

    debian01:
      ansible_host: 10.0.0.1
      ansible_user: user
      ansible_ssh_private_key_file: "~/.ssh/id_ed25519"
´´´
---
## Playbooks

For the playbooks, we will using them for assets hardening mainly, starting with Linux.

For my playbooks i will be setting up the following things:

    - UFW (deny all except ssh)
    - SSH only login without password
    - Fail2ban installation
