---
layout: default
title: Automating the Distribution of CA Certificates into NSS
collection: userconfig
permalink: userconfig/1_nss/
---
<**_Issue #20 includes these 2 problems that we are supposed to address: 
* Some of the intermediate CAs in the FPKI stop the CA name at OU rather than using a CN. What is the solution?
* Do full chains for client (user)-provided certificates need to be configured in the client for two-way TLS to succeed?_**>
-----------

Network Security Services (NSS) doesn't use the Windows Trust Store by default, so you'll need to manually add your CA certificates to the NSS Trust Store. This guide gives you an enterprise solution for automatically importing CA certificates into your NSS Trust Store.

* [Automate Importing CA Certificates into the NSS Trust Store](#automate-importing-ca-certificates-into-the-nss-trust-store-with-certutil)

## Configure Firefox To Use the Windows Trust Store <**This section is to be deleted**>

You can configure Firefox to use the Windows trust store<!--Is "file"="trust store"?-->: [Experimental Built-in Windows Support for Firefox Version 49 and Later](https://wiki.mozilla.org/CA:AddRootToFirefox){target="_blank"}_. You'll need to import the agency's root chain certificates into the Windows trust store.<!--??? "root chain certificates or "CA certificates" (title of doc.)? Shouldn't they already be in the Windows trust store? Unclear meaning.--> 

For Firefox to use the Windows trust store, you'll need to set the _security.enterprise_roots.enabled_ to _true_. For additional steps, go to:  [Firefox preference](https://developer.mozilla.org/en-US/docs/Mozilla/Preferences/A_brief_guide_to_Mozilla_preferences){target="_blank"}. You can make this change enterprise-wide.

{% include info-alert.hmtl content="If you set this preference on a client machines, the users won't be able to change it." %}

#### Client Machines <!--Enterprise-management solution needed per Issue #20 thread. Per LaChelle on 10/10, no JavaScripts .-->

You can create a JavaScript and put it into your users' Firefox profile directories so the JavaScript will run whenever a user launches Firefox.

```
JavaScript
// Set Firefox to trust the Windows trust file
lockPref("security.enterprise_roots.enabled", true);
```
* Restart Firefox.

#### Enterprise-wide 

You can also make this change across your agency's enterprise: [Enterprise Deployment of Firefox](https://developer.mozilla.org/en-US/Firefox/Enterprise_deployment){target="_blank"}_.

## Automate Importing CA Certificates into the NSS Trust Store with _Certutil_

Using _certutil_ is a good way to automatically import CA certificates into the NSS Trust Store.

### Prerequisites
<**Change the link name to "Add Certs to NSS Trust Store" or similar?**>
1. Install the NSS _certutil_ on your client machines. Go to: [Firefox-Add Certs](https://github.com/christian-korneck/firefox_add-certs/releases){target="_blank"}_. 
2. Client machines configured for PIV login.  

### Automate Importing CA Certificates into the NSS Trust Store

1. Using a Domain Controller, copy the CA certificate to the NSS directory so you can access it via _\\fileserver\scripts$\comp_resources\nss\publicca.cer_.
2. Open the Group Policy Management Console: _gpmc.msc_. 
3. Create and edit a Group Policy Object (GPO) using a test _OU_ (i.e., your target).<!--Is the test OU to solve the problem where CAs stop at "CA Name" rather than "OU" problem (LaChelle in original Issue)?-->
4. Navigate to User _Configuration\Policies\Windows Settings\Scripts\._ 
5. Double-click on _Logon_ and then click on _Show files_.
6. Create a new BAT file named _firefox_ca_add.bat_ that contains:  <**The preceding file name says "firefox." Change to just "ca_add.bat"? Also script below includes the word, "firefox," a couple of times.**>

            if not exist "%appdata%\mozilla\firefox\profiles" goto:eof
            set profiledir=%appdata%\mozilla\firefox\profiles
            dir "%profiledir%" /a:d /b > "%temp%\temppath.txt"
            if not exist "c:\windows\syswow64\nss" goto WIN32
            for /f "tokens=*" %%i in (%temp%\temppath.txt) do (
            cd /d "%profiledir%\%%i"
            copy cert8.db cert8.db.orig /y
            "c:\windows\syswow64\nss\certutil.exe" -A -n "Our Organization's Root CA" -i "c:\windows\system32\nss\publicca.cer" -t "TCu,TCu,TCu" -d . 
            )
            goto FINALLY
            :WIN32
            if not exist "c:\windows\system32\nss" goto FINALLY
            for /f "tokens=*" %%i in (%temp%\temppath.txt) do (
            cd /d "%profiledir%\%%i"
            copy cert8.db cert8.db.orig /y
            "c:\windows\system32\nss\certutil.exe" -A -n "Our Organization's Root CA" -i "c:\windows\system32\nss\publicca.cer" -t "TCu,TCu,TCu" -d . 
            )
            goto FINALLY
            :FINALLY
            del /f /q "%temp%\temppath.txt"

7. Go to the _Logon Properties_ window and click _Add_.
8. Browse to and double-click on the _firefox_ca_add.bat file_. <**Fix "firefox" - call just "ca_add.bat file"**>?
9. Double-click on _Logoff_ and go through Steps 5-8 again; however, this time use the directory tree to navigate to the BAT file (script) you created in Step 6 (e.g., C:\Users\<Username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup). 
10. Perform a _gpupdate /force_ on the test client machine and restart it. (You can also just run the BAT file.)
11. Open Firefox and go to _Tools_ **>** _Options_ **>** _Advanced_ **>** _Encryption_ tab **>** _Certificates_ pane. Click the _View Certificates_ button. <**This says to "open Firefox."**>
12. Scroll to [your organization’s] Root CA.
13. Remove an issued CA certificate: <**Does this final step make sense in terms of automatically importing CA certificates into the NSS Trust Store?**>

```

cd /d "%profiledir%\%%i"
"c:\windows\syswow64\nss\certutil.exe" -D -n "Our Organization's Root CA" -d 
```
