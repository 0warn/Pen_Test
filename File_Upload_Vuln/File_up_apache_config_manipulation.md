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

7. Use the back arrow in Burp Repeater or ciado replay to return to the original request for uploading your **PHP exploit**.
8. Change the value of the **filename parameter** from `exploit.php` to `exploit.l33t` or what extension name you give it `<.name>`. Send the request again and notice that the file was uploaded successfully.
	
9. Switch to the other Repeater or replay tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.l33t` and send the request. Observe that **Carlos's secret** was returned in the response. Thanks to our malicious `.htaccess` file, the `.l33t` file was executed as if it were a `.php` file. 
