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
> To Be Continue.......
