---
tags: CTFs
---
# OverTheWire's Natas - Challenges 20-29 Writeups
> Another run!

### Natas 20
Using the creds we got from Natas 19 we're met with Natas 20:
<img width="622" height="231" alt="image" src="https://github.com/user-attachments/assets/fb72c48a-2147-4b2f-b091-489fe021c9b7" /><br/>
Source code is as follows:
```
<?php

function debug($msg) { /* {{{ */
    if(array_key_exists("debug", $_GET)) {
        print "DEBUG: $msg<br>";
    }
}
/* }}} */
function print_credentials() { /* {{{ */
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas21\n";
    print "Password: <censored></pre>";
    } else {
    print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas21.";
    }
}
/* }}} */

/* we don't need this */
function myopen($path, $name) {
    //debug("MYOPEN $path $name");
    return true;
}

/* we don't need this */
function myclose() {
    //debug("MYCLOSE");
    return true;
}

function myread($sid) {
    debug("MYREAD $sid");
    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) {
    debug("Invalid SID");
        return "";
    }
    $filename = session_save_path() . "/" . "mysess_" . $sid;
    if(!file_exists($filename)) {
        debug("Session file doesn't exist");
        return "";
    }
    debug("Reading from ". $filename);
    $data = file_get_contents($filename);
    $_SESSION = array();
    foreach(explode("\n", $data) as $line) {
        debug("Read [$line]");
    $parts = explode(" ", $line, 2);
    if($parts[0] != "") $_SESSION[$parts[0]] = $parts[1];
    }
    return session_encode() ?: "";
}

function mywrite($sid, $data) {
    // $data contains the serialized version of $_SESSION
    // but our encoding is better
    debug("MYWRITE $sid $data");
    // make sure the sid is alnum only!!
    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) {
    debug("Invalid SID");
        return;
    }
    $filename = session_save_path() . "/" . "mysess_" . $sid;
    $data = "";
    debug("Saving in ". $filename);
    ksort($_SESSION);
    foreach($_SESSION as $key => $value) {
        debug("$key => $value");
        $data .= "$key $value\n";
    }
    file_put_contents($filename, $data);
    chmod($filename, 0600);
    return true;
}

/* we don't need this */
function mydestroy($sid) {
    //debug("MYDESTROY $sid");
    return true;
}
/* we don't need this */
function mygarbage($t) {
    //debug("MYGARBAGE $t");
    return true;
}

session_set_save_handler(
    "myopen",
    "myclose",
    "myread",
    "mywrite",
    "mydestroy",
    "mygarbage");
session_start();

if(array_key_exists("name", $_REQUEST)) {
    $_SESSION["name"] = $_REQUEST["name"];
    debug("Name set to " . $_REQUEST["name"]);
}

print_credentials();

$name = "";
if(array_key_exists("name", $_SESSION)) {
    $name = $_SESSION["name"];
}

?>
```
Interesting functions are seemingly `myread()` and `mywrite()` - session data is kept in some path unknown to us, `session_save_path()`. Admin probably has a session file for themselves we should find out.
We see `debug()` used in many parts of the code - including printing out `$filename` that we need. Appending `?debug` to the URL will show us whatever's printed:
<img width="556" height="245" alt="image" src="https://github.com/user-attachments/assets/4f818bff-c86a-4fd7-8070-2532fd48a355" /><br/>
So we now know sessions are saved under `/var/lib/php/sessions/` and with the prefix `mysess_` and then my SID. From this we can probably assume the admin's session is also saved there. Providing a different name each time as the output only overwrites my existing session's file, as we see the same SID is returned.
I wondered if I could insert some PHP code that will be saved under this file in order to create a webshell. I tried inserting the following as input:
```
<?php echo 'banans' ?>
```
Using `debug` again, this netted the following:
<img width="574" height="98" alt="image" src="https://github.com/user-attachments/assets/62333273-4467-4da8-9f54-d6b6ea4bd2c4" /><br/>
The length is still the entire command's length, but a value is missing in the name. Playing around with this for a while I couldn't figure out a way to use this to my adventure. Let's step back for a moment - I can also change the SID through my cookies, but we can use it for path traversal since characters other than the following are banned:
`1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-` 
I found the inclusion of `-` odd.
