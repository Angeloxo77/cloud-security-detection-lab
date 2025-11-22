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
### UFW Configuration Playbook
For this specific case, we are only allowing SSH access, but we could perfectly enable any other service/port, such as HTTP/HTTPS(80/443), mySQL(3306), 
```yaml
- name: Ensure package is installed
  apt:
    name: ufw
    state: present

- name: Allow SSH
  ufw:
    rule: allow
    port: 22

- name: Enable UFW
  ufw:
    state: enabled
    policy: deny
```
### SSH Configuration Playbook
We will disable SSH password authentication
```yaml
- name: Install SSH service if it is not by default
  ansible.builtin.package:
    name: openssh-server
    state: present

- name: Disallow SSH password authentication
  lineinfile:
    dest: /etc/ssh/sshd_config
    regexp: "^PasswordAuthentication"
    line: "PasswordAuthentication no"
    state: present
    validate: "sshd -t -f %s"

- name: Disable Root Login
  lineinfile:
       dest: /etc/ssh/sshd_config
       regexp: '^PermitRootLogin'
       line: "PermitRootLogin no"
       state: present
       backup: yes
  notify:
    - restart ssh
```
### Fail2ban Configuration Playbook
Installation of the package and copy config files.
```yaml
- name: Fail2ban package installation
  ansible.builtin.package:
    name: fail2ban
    state: present

- name: Copy Fail2ban .local file
  ansible.builtin.copy:
    src: "../files/jail.local"
    dest: "/etc/fail2ban/"
```
---
## Files

In the files section, we will be setting up default config files already configured to be secure.

In my case, I will be hardening the SSH protocol with some parametters:
```
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
backend = systemd
action = %(action_mwl)s

maxretry = 3
findtime = 600
bantime = 3600

[pam-generic]
enabled = true
filter = pam-generic
logpath = /var/log/auth.log
maxretry = 5
```
With this fail2ban will:
    - Enrich log files for future SIEM forward and parse (action).
    - Set 3 maximum tries for login.
    - Set a timer to find those login fails within 10 minutes (600 seconds).
    - Set a bantime of an hour to whoever fails the login 3 times within 10 minutes.
    - Block origin IP if theres an alert of 3 fails within 10 minutes.

In order to UnBan IPs we would need to do the following:

```bash
#Check for active jails
sudo fail2ban-client status
#Expected output:
#   Jail list: sshd

#Check for banned ips
sudo fail2ban-client status sshd
#Expected output:
#   Banned IP list: X.X.X.X X.X.X.X

#UnBan ip
sudo fail2ban-client set sshd unbanip X.X.X.X

#Check for ban reason
sudo zgrep "Ban" /var/log/fail2ban.log*
```
