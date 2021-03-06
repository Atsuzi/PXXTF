# Alcatel-Lucent Omnivista 4760/8770 RCE 0day
### Tldr
 * *4760* suffers an unauthenticated remote code execution as SYSTEM. No special configuration is required
 * *8770* and *4760* both suffer a remote administrative password disclosure. No special configuration required
 * *8770* suffer an authenticated remote code execution vulnerability. When chained with the disclosure vulnerability, it becomes an unauth RCE. In this case access to the port 389 and a directory license are required

## Previous work
* https://www.cvedetails.com/cve/CVE-2007-5190/
* https://github.com/malerisch/omnivista-8770-unauth-rce

## Intro
Alcatel OmniVista is a graphical interface to Alcatel OmniPCX, a common Voip solution. This software is used to manage the Voip accounts as well as to serve as a public directory. [Official product page](https://www.al-enterprise.com/en/products/communications-management-security/omnivista-8770-network-management-system).

I did notice this software a while ago while doing a penetration test. It caught my attention because the graphics interface looked somewhat old. As shown in the previous section, there wasn't any known vulnerability in this component. I wasn't unable to find any useful vulnerability without the source code, but only a few hints:

 * Exposed error log in `/log/error.log`
 * Error log showed LDAP errors when trying special characters in login and search forms
 * Error log showed failed `unserialize()` calls while decoding the `bookmarks`, the `themes` or the `cfilter` cookie

The LDAP injection didn't look too promising and while it *might* be possible to extract the hashed passwords I wasn't looking into that. I was more curious about the `unserialize()` calls but without any clue about the PHP code behind I decided to not waste any more time and try to get a copy of the whole software.

## Versions
 * *4760* very old and deprecated 
 * *8770* currently updated and sold

## The hardest part... Getting a copy
Unfortunately the OmniVista 4760/8770 software is not easy to get as a researcher. It is distributed as a CD/DVD only to legitimate companies via local Alcatel-Lucent partners.
I had a bit of experience in looking for not easily available softwares:
1. Get the file name (CD name) or any component name. It can be often found in online documentation, sometimes on the manufacturer website but most often on manuals uploaded by random users, for example on Scribd. Other resources includes dedicated forums, in this case [https://www.alcatelunleashed.com/](https://www.alcatelunleashed.com/).
2. Try to find a download, either by socializing on the forum, dorking in google for specific components (ie: intext:"ClientSetup.exe") or the full name. If possible write down a list of known versions and filenames and dork for any of them.
3. Look on 4shared.com :) https://www.4shared.com/rar/HsteugXy/A4760_R500702b.html (while the search function is not much powerful, sorting by size helps a lot)

## Unpacking it
Since the 4760 is an ancient product I fired up a Windows XP virtual machine and tried installing it. Like it happens most of the time with enterprise software, the installation failed and neither the main services were set up properly nor any PHP files was extracted. Part of the files were packed with the ancient ACE file format, probably with some custom modifications, and even binwalk couldn't do much.
When I started getting frustrated with all this old enterprise crap, i finally found the PHP files inside a CAB archive.


## Vulnerabilities

### 4760 pre auth RCE

_info.php_
```
<?php

require_once "vars.php";
$MyG["void"] = varform("void");
if ($MyG["void"] == "phDPhd") {
    phpinfo();
}

?>
```

The following two functions are used to get POST and GET variables as well as to manage SESSION.

_utils.php_
```
function varform($nom)
{
    if (!isset($_GET)) {
        global $MyG;
        if (isset($MyG[$nom])) {
            $var = $MyG[$nom];
            return $var;
        }
        $var = false;
        return $var;
    }
    if (isset($_GET[$nom])) {
        $var = $_GET[$nom];
        return $var;
    }
    if (isset($_POST[$nom])) {
        $var = $_POST[$nom];
        return $var;
    }
    $var = false;
    return $var;
}
function sessionform($nom)
{
    global $MyG;
    $toEncodeList = array("password", "ldappwd");
    if (!isset($MyG[$nom]) || $MyG[$nom] === false) {
        if (isset($_SESSION[$nom])) {
            if (in_array($nom, $toEncodeList)) {
                $var = decodepwd($_SESSION[$nom]);
                return $var;
            }
            $var = $_SESSION[$nom];
            return $var;
        }
        $var = false;
        return $var;
    }
    $var = $MyG[$nom];
    if (in_array($nom, $toEncodeList)) {
        $_SESSION[$nom] = encodepwd($var);
        return $var;
    }
    $_SESSION[$nom] = $var;
    return $var;
}
```
As we can see the following code checks for the users permissions before showing the page used to edit a template.

_EditThemeAction.php_
```
<?php
require_once "vars.php";
require_once "Action.php";
class EditThemeAction
{
    public function Invoke()
    {
        global $MyG;
        $MyG["themeId"] = varform("themeId");
        $MyG["themeId"] = sessionform("themeId");
        self::getaccessparameters();
        self::getuserlogin();
        $access = new CustomAccess($MyG["ldapHost"], $MyG["ldapPort"], "o=nmc", $MyG["userDn"], $MyG["userPass"]);
        if ($access->Connect() == true) {
            if ($access->TestAdminRights()) {
                $skin = new SkinAccess($MyG["themeId"]);
                $MyG["themeDate"] = $skin->GetLastMDate();
                sessionform("themeDate");
                $view = new EditThemeView($skin, false);
                $view->Display();
                $skin = null;
            } else {
                $view = new ErrorPopupView($access);
                $view->Display();
            }
            $access->Disconnect();
        } else {
            logerror(E_ERROR, $access->GetErrorMessage());
            $view = new ErrorPopupView($access);
            $view->Display();
        }
    }
}

?>
```

The default themes are numbered from 1 to 4 and each one has its files stored in `/theme/<id>`. Each theme folder contains a `params.st` file which contains a serialized PHP Object containing the theme configuration.
However, as seen below, the authentication and permission check is not performed when actually saving an edit. The only condition that might be a problem is the `CompareThemeDate()`, which compares the last edit time of the `params.st` file with the value saved in session in the code above (`$MyG["themeDate"] = $skin->GetLastMDate()`). This check, intended or not, prevents an unauthenticated user to do the save, unless in the destination folder a `params.st` file is not yet present.

_SaveThemeAction.php_
```
<?php

require_once "vars.php";
require_once "Action.php";
require_once "Access.php";
class SaveThemeAction
{
    public function Invoke()
    {
        global $MyG;
        logerror(E_NOTICE, "SaveThemeAction::Invoke()");
        $MyG["p"] = varform("p");
        $MyG["p"] = sessionform("p");
        $MyG["themeId"] = varform("themeId");
        $MyG["themeId"] = sessionform("themeId");
        $MyG["themeDate"] = sessionform("themeDate");
        $skin = new SkinAccess($MyG["themeId"]);
        if ($skin->CompareThemeDate($MyG["themeDate"])) {
            $skin->SetSkinParams($MyG["p"]);
            $skin->SetSkinImages($_FILES);
            $skin->BuildSkin();
            if ($skin->GetErrorNumber() == E_SUCCESS) {
                $MyG["themeDate"] = $skin->GetLastMDate();
                sessionform("themeDate");
                $themeAccess = new ThemeAccess();
                $themeAccess->SetThemeName($MyG["themeId"], $MyG["p"]["Name"]);
                $themeAccess = NULL;
            }
        }
        $view = new EditThemeView($skin, true);
        $view->Display();
        $skin = null;
    }
}

?>
```
The final pieces of vulnerable code are in

_SkinAccess.php_
```
class SkinAccess
{
	...
    public function __construct($idSkin)
    {
        $this->errno = E_SUCCESS;
        $this->idSkin = $idSkin;
        $this->path = "../Themes/Theme" . $this->idSkin . "/";
        $this->filename = $this->path . "params.st";
        $this->params["Id"] = $idSkin;
        $this->params["Name"] .= $idSkin;
        $this->LoadSkinParams();
    }
    ...

    public function CompareThemeDate($beforeDate)
    {
        $bEquals = true;
        $afterDate = $this->GetLastMDate();
        if ($beforeDate != $afterDate) {
            $bEquals = false;
            $this->errno = E_MOD_BYANOTHER;
        }
        return $bEquals;
    }

    public function SetSkinImages($files)
    {
        foreach ($files as $key => $file) {
            if ($file["error"] == 0) {
                if (strncmp($file["type"], "image", 5) != 0) {
                    $this->errno = E_BAD_IMAGETYPE;
                } else {
                    $this->params[$key] = $file["name"];
                    $outputFile = $this->path . $file["name"];
                    move_uploaded_file($file["tmp_name"], $outputFile);
                }
            }
        }
    }

```
So here it is! A path traversal in the `__construct()` and an unsecure file upload in  `SetSkinImages()`. Combining them we can set the upload folder to a folder which doesn't have yet a `params.st` file thus overcoming the `if ($skin->CompareThemeDate($MyG["themeDate"]))` condition because both values are empty!

The aforementioned exploit does not work in the 8770 because the path traversal is fixed, probably by checking that the `themeId` is between 1 and 4. The authentication issue seems to be still present, but the `CompareThemeDate()` check seems not possible to bypass.

### 4760/8770 LDAP admin credentials disclosure
The following PHP code is used to initialize some settings when a user starts a new session:
```
abstract class Action
{
	...
    public function GetAccessParameters()
    {
        global $MyG;
        $sess2reg = array("ldapHost" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Host", "ldapPort" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Port", "ldapSuDn" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Login", "ldapSuPass" => "HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\svc_mgr\\parameters\\LDAP Password");
        $MyG["ldapHost"] = sessionform("ldapHost");
        if ($MyG["ldapHost"] === false) {
            $shell = new COM("WScript.Shell");
            foreach ($sess2reg as $sess => $reg) {
                $MyG[$sess] = $shell->RegRead($reg);
                sessionform($sess);
                logerror(E_NOTICE, "Action::GetAccessParameters() [from registry] : " . $sess . "=" . $MyG[$sess]);
            }
            if (self::testledouxtruc() == false) {
                sessionunset("ldapHost");
                exit;
            }
        } else {
            foreach ($sess2reg as $sess => $reg) {
                $MyG[$sess] = sessionform($sess);
                logerror(E_NOTICE, "Action::GetAccessParameters() [from session] : " . $sess . "=" . $MyG[$sess]);
            }
        }
        self::getcompanysuffix();
    }
    ...
}

?>
```
So the code use the `COM` native PHP module to run some shell commands in order to get the LDAP bind credentials, which in this case are of "cn=directory manager" that is the administrator user of the instance. The password is encoded with a simple reversible algorithm we'll see below. Then the data is secured in the user session, which is stored server side.
While this code is bad, and the whole idea of putting the cleartext credentials in the registry doesn't make sense to me, the real problem is a webserver configuration: all user session files are stored in a public directory! So by just starting a session and getting the respective session file it is possible to get the credentials.
Sessions are stored in `/sessions/sess_<sessionid>`, simple as that.

The decode function:
```
function DecodePwd($data)
{
    $decryptData = "";
    if (strncasecmp("{NMC}", $data, 5) == 0) {
        $src = substr($data, 5);
    } else {
        $src = $data;
    }
    $a = 16;
    $len = strlen($src);
    for ($i = 0; $i < $len; $i++) {
        $c = substr($src, $i, 1);
        if (32 <= ord($c)) {
            $dst = ord($c) ^ $a;
            $b = chr($dst);
        } else {
            $dst = $c;
            $b = $c;
        }
        $decryptData .= $b;
        if (ord($b) != 0) {
            $a = $i * ord($b) % 255 >> 3;
        } else {
            $a = 16;
        }
    }
    return $decryptData;
}
```

### 8770 post auth RCE (to be verified)
By default, the installation also listens on port 389. By connecting to port 389 with the leaked credentials, one can edit the whole ldap tree including seeing and modifying the hashed password `AdminNmc` user which is the administrator of the PHP web interface. By using the newly obtained credentials it should not be a problem to upload a PHP file as an asset of an existing template.

Unfortunately, while all the previous vulnerabilities do work even when a "Directory License" (ndr the license specific for the PHP interface) is not present because the license check isn't done  as the first thing, this last one do not. It is possible to login and obtain a valid session with the leaked credentials, but it doesn't seem possible to get a valid `themeDate` in session.

Since I do not have access to the 8770 files and i can't test the upload code for the 8770.


## Other issues

 * Multiple calls to unserialize on untrusted data:

 	```
 	unserialize(gzuncompress($_COOKIE["themes"]));
 	unserialize(gzuncompress($_COOKIE["station"]));
 	unserialize(gzuncompress($_COOKIE["cfilter"]));
 	unserialize(gzuncompress($_COOKIE["bookmarks"]));
 	```
 I did not find an exploitable chain but: all the PHP version shipped with this product have multiple unserialize CVE and I did not find a way but it is possible to play with the COM class.

 * LDAP injections?

