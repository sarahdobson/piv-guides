---
layout: default
title: Automating the Distribution of CA Certificates into NSS
collection: userconfig
permalink: userconfig/1_nss/
---

Firefox doesn't use the Windows certificate store by default. This guide will help you to automate the distribution of Certification Authority (CA) certificates into the Firefox Network Security Services (NSS) trust store.

# Using Windows Certificate Store
You can configure Firefox ([version 49 and later](https://wiki.mozilla.org/CA:AddRootToFirefox){target="_blank"}_) to trust the Windows certificate store instead of its own NSS file. You have to import the agency specific root chain certificates in the Windows Certificate store.

In order for Firefox to use the Windows trust store, you have to update the [Firefox preference](https://developer.mozilla.org/en-US/docs/Mozilla/Preferences/A_brief_guide_to_Mozilla_preferences){target="_blank"}_ “security.enterprise_roots.enabled” to ‘true’. You can update this preference using a javascript file created on each target machine.

```javascript
// Set Firefox to trust the Windows trust file
lockPref("security.enterprise_roots.enabled", true);
```

Place this javascript file in the users firefox profile directory so that it is picked up everytime Firefox starts up. Note that the preference is locked in the example above and the user cannot modify this preference. You can choose other options to deploy this in an [enterprise deployment of Firefox](https://developer.mozilla.org/en-US/Firefox/Enterprise_deployment){target="_blank"}_.

# Using CertUtil

## Prerequisites

1. Install the Firefox NSS _certutil_ on your client machines. Go to: [Firefox-Add Certs](https://github.com/christian-korneck/firefox_add-certs/releases){target="_blank"}_.
2. Configure your client machines for PIV login.  

## Automate the Distribution of CA Certificates into the NSS Trust Store

1. Using a Domain Controller, copy the CA certificate to the NSS directory so you can access it via _\\fileserver\scripts$\comp_resources\nss\publicca.cer_.
2. Open the Group Policy Management Console: _gpmc.msc_. 
3. Create and edit a Group Policy Object (GPO) using a test _OU_ (i.e., your target).
4. Navigate to User _Configuration\Policies\Windows Settings\Scripts\._ 
5. Double-click on _Logon_ and then click on _Show files_.
6. Create a new BAT file named _firefox_ca_add.bat_ that contains: 

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

7. Go to the Logon Properties window and click _Add_.
8. Browse to and double-click on the _firefox_ca_add.bat file_.
9. Double-click on _Logoff_ and go through Steps 5-8 again; however, this time use the directory tree to navigate to the BAT file (script) you created in Step 6 (e.g., C:\Users\<Username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup). 
10. Perform a _gpupdate /force_ on the test client machine and restart it. (You can also just run the BAT file.)
11. Open Firefox and go to _Tools_ **>** _Options_ **>** _Advanced_ **>** _Encryption_ tab **>** _Certificates_ pane. Click the _View Certificates button_. 
12. Scroll to [your organization’s] Root CA.
13. Remove an issued CA certificate: 

```

cd /d "%profiledir%\%%i"
"c:\windows\syswow64\nss\certutil.exe" -D -n "Our Organization's Root CA" -d 
```
