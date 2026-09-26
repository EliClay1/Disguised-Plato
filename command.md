```bash
sudo -i
date -u
who -a
w
last -Fai | head -n 50
lastb -Fai 2>/dev/null | head -n 50
ss -lntup
ss -antup
ps auxf
systemctl --failed --no-pager
systemctl --type=service --state=running --no-pager
systemctl list-timers --all --no-pager
getent passwd
getent group sudo 2>/dev/null
getent group wheel 2>/dev/null
find /root /home -xdev -type f -name authorized_keys -ls 2>/dev/null
journalctl -u ssh -u sshd --since '-2 hours' --no-pager
```

```bash
apt-get update && apt-get install -y auditd audispd-plugins
systemctl enable auditd
service auditd start
systemctl status auditd --no-pager
auditctl -s
auditctl -l
```

```bash
sudo dnf install -y audit audispd-plugins
sudo systemctl enable auditd
sudo service auditd start
sudo systemctl status auditd --no-pager
sudo auditctl -s
sudo auditctl -l
```

```bash
TS=$(date -u +%Y%m%dT%H%M%SZ)
install -d -m 700 /root/ccdc-audit-backup
cp -a /etc/audit /root/ccdc-audit-backup/audit-$TS
```

```bash
cat > /etc/audit/rules.d/99-ccdc.rules <<'RULES'
-w /etc/passwd -p wa -k identity_changes
-w /etc/group -p wa -k identity_changes
-w /etc/shadow -p wa -k identity_changes
-w /etc/gshadow -p wa -k identity_changes
-w /etc/sudoers -p wa -k privilege_changes
-w /etc/sudoers.d/ -p wa -k privilege_changes
-w /etc/ssh/sshd_config -p wa -k ssh_changes
-w /root/.ssh/ -p wa -k ssh_key_changes
-w /etc/systemd/system/ -p wa -k persistence_changes
-w /etc/init.d/ -p wa -k persistence_changes
-w /etc/rc.local -p wa -k persistence_changes
-w /etc/crontab -p wa -k persistence_changes
-w /etc/cron.d/ -p wa -k persistence_changes
-w /etc/cron.daily/ -p wa -k persistence_changes
-w /etc/cron.hourly/ -p wa -k persistence_changes
-w /etc/cron.weekly/ -p wa -k persistence_changes
-w /etc/cron.monthly/ -p wa -k persistence_changes
-w /var/spool/cron/ -p wa -k persistence_changes
-w /etc/nginx/ -p wa -k nginx_changes
-w /var/www/ -p wa -k web_changes
-a always,exit -F arch=b64 -S execve -F euid=0 -k privileged_exec
-a always,exit -F arch=b32 -S execve -F euid=0 -k privileged_exec
RULES
augenrules --load
auditctl -l
service auditd restart
systemctl status auditd --no-pager
```

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
journalctl --flush
journalctl --disk-usage
```

```bash
TS=$(date -u +%Y%m%dT%H%M%SZ)
install -d -m 700 /root/ccdc-baseline
tar --xattrs --acls -czf /root/ccdc-baseline/nginx-web-$TS.tgz /etc/nginx /var/www 2>/dev/null
sha256sum /root/ccdc-baseline/nginx-web-$TS.tgz > /root/ccdc-baseline/nginx-web-$TS.tgz.sha256
find /etc/nginx /var/www -xdev -type f -exec sha256sum {} + | sort -k2 > /root/ccdc-baseline/nginx-web-files-$TS.sha256
nginx -t
nginx -T > /root/ccdc-baseline/nginx-effective-$TS.conf 2>&1
curl -i http://127.0.0.1/
```

```bash
SPLUNK_HOME=/opt/splunkforwarder
SPLUNK_USER=$(stat -c '%U' "$SPLUNK_HOME" 2>/dev/null)
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" status
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" btool check
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" btool outputs list --debug
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" btool inputs list --debug
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" list forward-server
```

```bash
SPLUNK_HOME=/opt/splunkforwarder
SPLUNK_USER=$(stat -c '%U' "$SPLUNK_HOME")
install -d -o "$SPLUNK_USER" -g "$SPLUNK_USER" -m 750 "$SPLUNK_HOME/etc/apps/ccdc_monitor/local"
cat > "$SPLUNK_HOME/etc/apps/ccdc_monitor/local/inputs.conf" <<'INPUTS'
[monitor:///var/log/audit/audit.log]
disabled = false
index = main
sourcetype = linux:audit

[monitor:///var/log/auth.log]
disabled = false
index = main
sourcetype = linux_secure

[monitor:///var/log/secure]
disabled = false
index = main
sourcetype = linux_secure

[monitor:///var/log/nginx/access.log]
disabled = false
index = main
sourcetype = nginx:plus:access

[monitor:///var/log/nginx/error.log]
disabled = false
index = main
sourcetype = nginx:plus:error
INPUTS
chown -R "$SPLUNK_USER:$SPLUNK_USER" "$SPLUNK_HOME/etc/apps/ccdc_monitor"
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" btool check
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" restart
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" list forward-server
```

```bash
watch -n 2 'date -u; echo === USERS ===; w; echo === CONNECTIONS ===; ss -antup | head -n 100'
tail -F /var/log/auth.log /var/log/secure /var/log/audit/audit.log /var/log/nginx/access.log /var/log/nginx/error.log 2>/dev/null
journalctl -f -u ssh -u sshd -u nginx
```

```bash
ausearch -ts recent -m USER_LOGIN,USER_AUTH,USER_START,USER_END -i
ausearch -ts recent -k identity_changes -i
ausearch -ts recent -k privilege_changes -i
ausearch -ts recent -k ssh_changes -i
ausearch -ts recent -k ssh_key_changes -i
ausearch -ts recent -k persistence_changes -i
ausearch -ts recent -k nginx_changes -i
ausearch -ts recent -k web_changes -i
ausearch -ts recent -k privileged_exec -i
aureport -au -ts today -i
aureport -l -ts today -i
aureport -x -ts recent -i | tail -n 100
journalctl -u ssh -u sshd --since '-30 minutes' --no-pager
find /etc/nginx /var/www -xdev -type f -mmin -30 -printf '%TY-%Tm-%Td %TH:%TM:%TS %u %g %m %p\n' | sort
```

```bash
# LINUX — 11. CHECK CURRENT PERSISTENCE AND RECENT FILE CHANGES
systemctl list-unit-files --state=enabled --no-pager
systemctl list-timers --all --no-pager
find /etc/cron* /var/spool/cron -maxdepth 3 -type f -ls 2>/dev/null
find /etc/systemd /usr/lib/systemd /lib/systemd -type f -mmin -60 -ls 2>/dev/null
find /tmp /var/tmp /dev/shm -xdev -type f -ls 2>/dev/null
find /var/www -xdev -type f -mmin -60 -ls 2>/dev/null
lsof +L1 2>/dev/null
```

```bash
mkdir -p /root/ccdc-pcap
chmod 700 /root/ccdc-pcap
tcpdump -i any -nn -s 128 -C 50 -W 4 -w /root/ccdc-pcap/connections.pcap 'tcp or udp'
```

```powershell
Get-Date -Format o
quser.exe
qwinsta.exe
Get-NetTCPConnection | Sort-Object State,RemoteAddress,RemotePort
Get-NetUDPEndpoint | Sort-Object LocalPort
Get-NetTCPConnection -State Listen | ForEach-Object {$p=Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue; [pscustomobject]@{Address=$_.LocalAddress;Port=$_.LocalPort;PID=$_.OwningProcess;Process=$p.ProcessName}}
Get-LocalUser
Get-LocalGroupMember Administrators
Get-Service | Sort-Object Status,Name
Get-ScheduledTask | Select-Object TaskPath,TaskName,State,Author
Get-CimInstance Win32_Service | Select-Object Name,State,StartMode,StartName,PathName
```

```powershell
$TS=(Get-Date).ToUniversalTime().ToString('yyyyMMddTHHmmssZ')
$Root="C:\CCDC-Private\Audit-$TS"
New-Item -ItemType Directory -Path $Root -Force | Out-Null
icacls $Root /inheritance:r /grant:r 'Administrators:(OI)(CI)F' 'SYSTEM:(OI)(CI)F'
auditpol.exe /backup /file:"$Root\audit-policy.csv"
auditpol.exe /set /subcategory:'Logon' /success:enable /failure:enable
auditpol.exe /set /subcategory:'Logoff' /success:enable
auditpol.exe /set /subcategory:'Special Logon' /success:enable
auditpol.exe /set /subcategory:'Process Creation' /success:enable /failure:enable
auditpol.exe /set /subcategory:'User Account Management' /success:enable /failure:enable
auditpol.exe /set /subcategory:'Security Group Management' /success:enable /failure:enable
auditpol.exe /set /subcategory:'File System' /success:enable /failure:enable
auditpol.exe /set /subcategory:'Other Object Access Events' /success:enable /failure:enable
reg.exe add 'HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit' /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
auditpol.exe /get /category:*
```

```powershell
$Path='C:\inetpub\wwwroot'
$Acl=Get-Acl $Path
$Rule=New-Object System.Security.AccessControl.FileSystemAuditRule('Everyone','CreateFiles,WriteData,AppendData,Delete,ChangePermissions,TakeOwnership','ContainerInherit,ObjectInherit','None','Success,Failure')
$Acl.AddAuditRule($Rule)
Set-Acl -Path $Path -AclObject $Acl
(Get-Acl $Path).Audit
```

```powershell
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' -Force | Out-Null
New-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' -Name EnableScriptBlockLogging -PropertyType DWord -Value 1 -Force | Out-Null
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging' -Force | Out-Null
New-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging' -Name EnableModuleLogging -PropertyType DWord -Value 1 -Force | Out-Null
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging\ModuleNames' -Force | Out-Null
New-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging\ModuleNames' -Name '*' -PropertyType String -Value '*' -Force | Out-Null
```

```powershell
wevtutil.exe sl Security /ms:268435456
wevtutil.exe sl System /ms:134217728
wevtutil.exe sl 'Microsoft-Windows-PowerShell/Operational' /e:true /ms:134217728
wevtutil.exe sl 'Microsoft-Windows-TerminalServices-LocalSessionManager/Operational' /e:true /ms:67108864
wevtutil.exe sl 'Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational' /e:true /ms:67108864
```

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4624,4625,4634,4648,4672;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-TerminalServices-LocalSessionManager/Operational';Id=21,23,24,25;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational';Id=1149;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4688;StartTime=(Get-Date).AddMinutes(-30)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4720,4722,4724,4728,4732,4756;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4656,4663,4670;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4697,4698,4699,4702;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='System';Id=7045;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational';Id=4103,4104;StartTime=(Get-Date).AddHours(-2)} | Select-Object TimeCreated,Id,Message
```

```powershell
while ($true) {Clear-Host; Get-Date; quser.exe; Get-NetTCPConnection -State Established | Sort-Object RemoteAddress,RemotePort | Format-Table -AutoSize; Start-Sleep 2}
```

```powershell
$TS=(Get-Date).ToUniversalTime().ToString('yyyyMMddTHHmmssZ')
$Root="C:\CCDC-Private\Evidence-$TS"
New-Item -ItemType Directory -Path $Root -Force | Out-Null
wevtutil.exe epl Security "$Root\Security.evtx"
wevtutil.exe epl System "$Root\System.evtx"
wevtutil.exe epl 'Microsoft-Windows-PowerShell/Operational' "$Root\PowerShell.evtx"
wevtutil.exe epl 'Microsoft-Windows-TerminalServices-LocalSessionManager/Operational' "$Root\RDP-LocalSession.evtx"
Get-NetTCPConnection | Export-Csv "$Root\connections.csv" -NoTypeInformation
Get-CimInstance Win32_Service | Export-Csv "$Root\services.csv" -NoTypeInformation
Get-ScheduledTask | Export-Clixml "$Root\tasks.xml"
```

```text
show users
show log
show log | match ssh
show conntrack table ipv4
show configuration commands
show system commit
monitor log
```

```bash
SPLUNK_HOME=/opt/splunk
SPLUNK_USER=$(stat -c '%U' "$SPLUNK_HOME")
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" status
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" btool check
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" btool inputs list --debug
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" btool outputs list --debug
sudo -u "$SPLUNK_USER" "$SPLUNK_HOME/bin/splunk" show listen
```

```text
index=* earliest=-30m ("Accepted password" OR "Accepted publickey" OR "session opened" OR EventCode=4624 OR EventCode=4625)
| table _time host source user src_ip EventCode message
| sort - _time
```

```text
index=* earliest=-30m (key=identity_changes OR key=privilege_changes OR key=persistence_changes OR key=nginx_changes OR key=web_changes OR EventCode IN (4663,4697,4698,4702,4720,4728,4732,4756,7045))
| table _time host source user key EventCode process file path message
| sort - _time
```

```text
index=* earliest=-30m (key=privileged_exec OR EventCode=4688 OR EventCode=4103 OR EventCode=4104)
| table _time host user process process_name process_path process_command_line CommandLine Message
| sort - _time
```

```text
| metadata type=hosts
| eval age_seconds=now()-recentTime
| sort - age_seconds
```
****
