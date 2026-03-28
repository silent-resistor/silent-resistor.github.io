---
title: "Phishing"
date: 2026-03-28
weight: 2
---


## Intro to Phishing..
- **Social Engineering** is the psychological manipulation of people into performing or divulging information by exploiting weakness in human nature. These "weakness" can be curiosity, jealously, greed, and even kindness and the willingness to help someone. 
- **Phishing** is a source of social engineering delivered through email to trick someone into either revealing personal information, credentials, or even executing malicious code on their computer.
- These emails will usually appear to come from a trusted source, whether that's person or a business. They include content that tries to tempt or trick people into downloading software, opening attachments, opening attachements, or following links to bogus website.
- A term you'll come across and the type of phishing campaign a red team would participate in is **spear-phishing**, as with throwing a physical spear; you'd have a target to aim at, the same can be said with spear-phishing in that you're targeting an individual, business or organization rather than just anybody as mass. This is an effective form of phishing for a red team engagement as they are bespoke to the target it makes them  hard to detect by technology such as spam filters, antivirus, and firewalls.
- A red team could be contracted to solely carry out a phishing assessment to see whether a business is vulnerable to this type of attack or can also be part of a broader scale assessment and used to gain access to computer systems or services.
- Some other methods of phishing through other mediums are smishing which is phishing through SMS messages, and vishing which is performed through phone calls.

## Writing Convincing Phishing Emails
- We have three things to work with regarding phishing emails: the sender's email, the subject and the content
- **Sender's Address:** Ideally it would be from domain name that spoofs a significant branch, a known contact, or a co-worker.
  - To find what brands or people a victim interacts, you can employ OSINT tactics. For example
  - Observe their social media account for any brands or friends they talk to 
  - Searching Google for the victim's name and rough location for any reviews the victim may have left about local business or brands.
  - Looking at the victim's bussiness webisite to find suppliers
  - Looking at LinkedIn to find coworkers fo the victim.

- **The Subject line:** We should set the subject to something quite urget, worrying, or piques the victim's curiosity, so they do not ignore it and act on it quickly.
  - Lets say examples as "Your account has been comprised", "Your package has been dispatched", "Staff payroll information (do not forward)", "Your photos have been published".
  
- **The Content:** 
  - If impersonating a brand or supplier, it would be pertinent to research their standard email templates and branding (style, logo's images, signoffs etc.) and make content look same as theirs, so the victim doesnt expect anything. 
  - If impersonating the contact or co-worker, it could be beneficial to contact them; first they may have some branding in their templete, have a particular email signature or even something small such as how they refer to themselves, for example, someone might have the name Dorothy and their emial is dorothy@company.thm. Still, in their signature, it might say "Best Regards, Dot". Learning these somewhat small things can sometimes have quite dramatic psychological effects on the victim and convice them more to open and act on the email.
  - If you've set up a sppof website to harvest data or distribute the malware, the links to this should be disuised using `anchor text` and changing it either some text which say **Clik here** or changing it to a correct that reflects the businees you are spoofing, for example
  - `<a href="http://spoofsite.thm">Click here</a>`
  - `<a href="http://spoofsite.thm">https://onlineback.thm</a>`



## Phishing Infrastructure
- A certain amount of infrastructure will need to be put in place to launch a successfull phishing campaign.
- **Domain Name:**
  - We'll need to register either an authentic-looking domain name or one that mimics the identity of another domain. 
- **SSL/TLS Certificates:**
  - Creating SSL/TLS certificates for you chosen domain name will add an extra layer of authenticity to the attack.
- **Email Server/Account:**
  - You'll need to either to setup an email server or register with an SMTP email provider.
- **DNS Records:**
  - Setting up DNS records such as SPF,DKIM, DMARC will improve the deliverability of your emails and make sure they're getting into the inbox rather than spam folder.
- **Web server:**
  - Setting up webserver or purchase web hosting from a company to host your phishing websites. Adding SSL/TLS to the websites will give them an extra layer of authenticity
- **Analytics:**
  - When a phishing campaign is part of a red team engagement, keeping analytics information is more important. You'll need to keep track of the emails that been sent, opened or clicked. You'll also need to combine it with the information from your phishing websites for which users have supplied personal information or downloaded software.
- **Automation and usefull software:**
  - - Some of the above infrastructure can be quickly automated by using below tools:
  - **GoPhish - (Open source phishing framework) - getgophish.com**
    - Its web-based framework to make setting up phishing campaigns more straight forward. 
    - Gophish allows you to store your smtp server settings for sending emails, has a web-based tool fro creating emails templates using a simple WYSIWYG (What You See is What you Get) editor. 
    - You can also schedule when emails are sent and have an analytics dashboard that show how many emails have sent, opened or clcked.
  - **SET - (Social Engineering Toolkit) - trustedsec.com**
    - The SET contains a multinode of tools, but some of the important ones for phishing are the ability to create spear-phishing attacks and deploy fake versions of common websites to trick victims into entering their credentials.

## Droppers
- Droppers are software that phishing victims tend to be tricked into downloading and running on their system. The dropper may advertise itself as something usefull or legitimate such as codec to view a certain video or software to open a specific file.
- Droppers are not usually malicious themselves, so they tend to pass antivirus checks. Once installed, the intended malware is either upacked or downloaded from a server and installed onto the victim's system. The malicious softwares usually connects back to the attacker's infracture. The attacker can take control of the victim's computer, which further explore and exploit the local network.


## Choosing A phishing Domain
- Choosing the right phishing domain to launch attack from is essential to ensure you have the psychological edge over target. A red team engagement can use some of the below methods for choosing the perfect domain name.
- **Expired Domains:**
  - Although not essential, buying a domain name with some history may lead to better scoring of your domain when it comes to spam filters. Spam filters have a tendency to not trust brand new domain names compared to one with some history.
- **Typosquatting:**
  - It is when a registered domain looks very similar to the target domain you're trying to impersonate. 
- **TLD Alternatives:**
  - A TLD (Top Level Domain) is the .com, .org, .co.uk, .gov, etc. are part of a domain name, there are 100's of varients of TLD's now. A common trick for choosing a domain would be to use the same name but with different TLD.
- **IDN Homograph Attack/Script Spoofing:**
  - Originally domain names were made up of Latin characters a-z, and 0-9, but in 1998, IDN (International domain name) was implemented to support language specific script or alphabet from other languages such as Arabic, Chinese, Cyrillic, etc.
  - An issue that arises from the IDN implementation is that different languages can actually appear identical. For example, Unicode character U+0430 (Cyrillic small letter a) looks identical to Latin character a (U+0061) used in english, enabling attackers to register a domain name that looks more identical to another.

## Using MS Office in Phishing
- Often during phishing campaigns, a MS Office document is included as attachment. Office documents can contain macros, which do have a legitimate use but also can be used to run computer commands that can cause malware to be installed onto the victim's computer or connect back to an attacker's network and allow to take control of the his computer.



## Using Browser Exploits
- Another method of gaining control over a victim's computer could be browser exploits;; this is when there is a vulnerability against a browser itself (Internet Explorer/Edge, Chrome, Safari etc), which allows the attacker to run remote commands on the victim's computer.
- Browser exploits aren't usually a common path to follow in a red team engagement unless you have prior knowledge of old technology being used onsite. Many browsers are kept upto date, hard to exploit due to how browsers are developed, and exploits are often worth a lot of money if reported back to the developers.
- That being said, it can happen, and as previously mentioned, it could be used to target old technologies on-site because possibly the browser software cannot be updated due to compatibility with commercial software/hardware, which can happen quite often in big institutions such as education, government and especiallyl health care.
- Usually, the victim would receive an email, convincing them to visit a paritcular webisite setup by the attacker. Once teh victim is on the site, the exploit works against the browser, and now the attacker can perform any commands they wish on the victim's computer.




