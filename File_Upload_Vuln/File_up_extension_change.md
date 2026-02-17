## Insufficient blacklisting of dangerous file types

One of the more obvious ways of preventing users from uploading malicious scripts is to blacklist potentially dangerous file extensions like `.php`. The practice of blacklisting is inherently flawed as it's difficult to explicitly block every possible file extension that could be used to execute code. Such blacklists can sometimes be bypassed by using lesser known, alternative file extensions that may still be executable, such as `.php5`, `.shtml`, and so on.

---

## Obfuscating file extensions

- Even the most exhaustive blacklists can potentially be bypassed using classic obfuscation techniques. Let's say the validation code is case sensitive and fails to recognize that exploit.pHp is in fact a .php file. If the code that subsequently maps the file extension to a MIME type is not case sensitive, this discrepancy allows you to sneak malicious PHP files past validation that may eventually be executed by the server.

- You can also achieve similar results using the following techniques:

	1. Provide multiple extensions. Depending on the algorithm used to parse the filename, the following file may be interpreted as either a `PHP` file or `JPG image`: `exploit.php.jpg` Add trailing characters.
 	2. Some components will strip or ignore trailing whitespaces, dots, and suchlike: `exploit.php.`
  	3. Try using the URL encoding (or double URL encoding) for dots, forward slashes, and backward slashes. If the value isn't decoded when validating the file extension, but is later decoded server-side, this can also allow you to upload malicious files that would otherwise be blocked: `exploit%2Ephp`
  	4. Add semicolons or URL-encoded null byte characters before the file extension. If validation is written in a high-level language like **PHP** or **Java**, but the server processes the file using lower-level functions in **C/C++**, for example, this can cause discrepancies in what is treated as the end of the filename: `exploit.asp;.jpg` or `exploit.asp%00.jpg`
  	5. Try using multibyte unicode characters, which may be converted to null bytes and dots after unicode conversion or normalization. Sequences like xC0 x2E, xC4 xAE or xC0 xAE may be translated to x2E if the filename parsed as a UTF-8 string, but then converted to ASCII characters before being used in a path.

--- 

## Obfuscating file extensions - Continued

- Other defenses involve stripping or replacing dangerous extensions to prevent the file from being executed. If this transformation isn't applied recursively, you can position the prohibited string in such a way that removing it still leaves behind a valid file extension. For example, consider what happens if you strip .php from the following filename:

```PHP
	exploit.p.phphp
```
This is just a small selection of the many ways it's possible to obfuscate file extensions.

## PROCESS

1. Log in and upload an image as your avatar, then go back to your account page.
2. In Burp or caido, go to `Proxy > HTTP history` and notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater  or caido replay.

3. On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret. For example:
```PHP
	<?php echo file_get_contents('/home/carlos/secret'); ?>
					OR
	<?php system ($_GET['cmd']); ?>
```
4. Attempt to upload this script as your avatar. The response indicates that you are only allowed to upload `JPG` and `PNG` files.
5. In Burp's or caido's proxy history, find the `POST /my-account/avatar` request that was used to submit the file upload. Send this to Burp Repeater or caido replay.
6. In Burp Repeater or caido replay, go to the tab for the `POST /my-account/avatar` request and find the part of the body that relates to your `PHP` file. In the Content-Disposition header, change the value of the filename parameter to include a URL encoded null byte, followed by the `.jpg` extension:
```FILENAME-CHANGING
	filename="exploit.php%00.jpg"
```

7. Send the request and observe that the file was **successfully** uploaded. Notice that the message refers to the file as `exploit.php`, suggesting that the null byte and `.jpg` extension have been **stripped**

8. Switch to the other Repeater tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.php` and send the request. Observe that Carlos's secret was returned in the response.

9. Submit the secret to solve the lab.

--- 

--- 
