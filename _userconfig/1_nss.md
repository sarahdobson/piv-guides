---
layout: default
title: Automating the Distribution of CA Certificates into NSS
collection: userconfig
permalink: userconfig/1_nss/
---
If you want to automate importing CA intermediate certificates into Network Security Service (NSS) for use by Firefox, these are the steps to follow. <!--Needs more government context. Is the admin automating the importing of CA intermedicate certs for the users' benefit so they don't have to do it it manually or have a problem with getting an error like "certificate expired"? Is this NSS = the "FIPS-Mode" NSS?--> 

## Prerequisites

1. Install the Network Security Services (NSS) _certutil_ on your client machines. Go to: [Firefox-Add Certs](https://github.com/christian-korneck/firefox_add-certs/releases){target="_blank"}_.
2. Configure your client machines for PIV login. <!--Should we say "PIV login with Firefox"? We should include a link to the new Firefox Playbook when LaChelle moves Firefox Playbook to Staging.-->
<!--? Do we need to specify for the Domain Controller what Windows Server Releases are needed?--> 

## Create a Script To Distribute CA Certificates to NSS

1. Using a Domain Controller, copy the CA certificate to the NSS directory so you can access it via _\\fileserver\scripts$\comp_resources\nss\publicca.cer_.
2. Open the Group Policy Management Console: _gpmc.msc_. <!--If the admin is using a Windows Server R[x] to get to gpmc.msc, what Server version should he/she be using?  List as a Prerequisite?-->
3. Create and edit a Group Policy Object (GPO) using a test _OU_ (i.e., your target).
4. Navigate to _User Configuration\Policies\Windows Settings\Scripts\_._ 
5. Double-click on _Logon_ and then click on _Show files_.
6. Right-click and create a new BAT file named _firefox_ca_add.bat_ that contains: <!--Right-click on what, to do what? Is the BAT file the "script" the admin "added to the "/Startup/ directory" mentioned in Step 9? Explain "/Startup/ directory.-->

            if not exist "%appdata%\mozilla\firefox\profiles" goto:eof
            set profiledir=%appdata%\mozilla\firefox\profiles
            dir "%profiledir%" /a:d /b > "%temp%\temppath.txt"
            if not exist "c:\windows\syswow64\nss" goto WIN32
            for /f "tokens=*" %%i in (%temp%\temppath.txt) do (
            cd /d "%profiledir%\%%i"
            copy cert8.db cert8.db.orig /y
            "c:\windows\syswow64\nss\certutil.exe" -A -n "Our Organization's Root CA" -i "c:\windows\system32\nss\publicca.cer" -t "TCu,TCu,TCu" -d . <!--The space before period?-->
            )
            goto FINALLY
            :WIN32
            if not exist "c:\windows\system32\nss" goto FINALLY
            for /f "tokens=*" %%i in (%temp%\temppath.txt) do (
            cd /d "%profiledir%\%%i"
            copy cert8.db cert8.db.orig /y
            "c:\windows\system32\nss\certutil.exe" -A -n "Our Organization's Root CA" -i "c:\windows\system32\nss\publicca.cer" -t "TCu,TCu,TCu" -d . <!--The space before period is correct?-->
            )
            goto FINALLY
            :FINALLY
            del /f /q "%temp%\temppath.txt"

7. Go to the Logon Properties window and click _Add_.
8. Browse to and double-click on the _firefox_ca_add.bat file_.
9. Double-click on _Logoff_ and go through Steps 5-8 again, but using the directory tree, navigate to the script you created in the _./Startup/_ directory (e.g., for Windows 10:  C:\Users\<Username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup). <!--Is this script the same as the BAT file from Step 6 (unclear)? Is this pathname for Windows Server Release x? vs. Windows 10?-->
10. Perform a _gpupdate /force_ on the test client machine and restart it. (You can also just run the BAT file.) <!--Up until this step, the admin is doing these steps on the Domain Controller?  Is this Step (10) being done via remote access from the Domain Controller to the test client machine?-->
11. Open Firefox and go to _Tools_ **>** _Options_ **>** _Advanced_ **>** _Encryption_ tab **>** _Certificates_ pane. Click the _View Certificates button_. <!--Is admin doing Steps 11-13 from the Domain Controller?-->
12. Scroll to [your organization’s] Root CA.
13. Remove an issued CA certificate: <!--Can't follow the logic of this ending. What does this have to do with "automating distribution of CA intermediate certificates into NSS"? Is the removed certificate = the "Root CA" in Step 12? Needs to be clarified and tied into the purpose/context for this Playbook.-->

```

cd /d "%profiledir%\%%i"
"c:\windows\syswow64\nss\certutil.exe" -D -n "Our Organization's Root CA" -d 
```
