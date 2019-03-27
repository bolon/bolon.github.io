---
layout: post
title: Setup Local Secured Service (TLS/SSL) for Android App Testing
---

### Some context
This problem surfaced when I need to mock endpoints for an Android Application,  
the application allow it's user to change the endpoint from user interface but only accept TLS connection.

There are 2 options crossed my mind at that time:
  1. Change hardcoded prefix `https` for endpoint
     It turned out that piece of codes to mock the endpoint is tightly coupled to other module, and the effort & risk were too much for this task.
  2. Setup middleware to route my port `443` to local mock server.
     So I choose this solution as it seems plausible and doesn't touch existing code.

### Steps
  1. Initially [mock-server](http://www.mock-server.com/) already up and running on docker container and talk to host on port `1080`.
  2. Then set up webserver on my local machine using [Nginx](https://www.nginx.com/resources/wiki/start/)
  3. For TLS connection we need certificate.
     Since this only for testing purpose I decided to use self-signed certificate.
     Make sure the generated certificate also installed on Android so the handshake can be done properly.
     To generate the certificate use these step:
      - Create an `android_options.txt` file with this line of code
          `basicConstraints=CA:true`
      - `openssl genrsa -out priv_and_pub.key 2048`  -> 2048 number of bits should be sufficient.
      - `openssl req -new -days 3650 -key priv_and_pub.key -out CA.pem`
         When generating this you will be asked with some information, but the important one is the `Common Name`,
        `Common Name` is a Fully Qualified Domain Name [(FQDN)](https://en.wikipedia.org/wiki/Fully_qualified_domain_name).
        For example I set `Common Name : mock-api.com`. Later I will show how to resolve this domain name to your local address.
      - `openssl x509 -req -days 3650 -in CA.pem -signkey priv_and_pub.key -extfile ./android_options.txt -out CA.crt`
      - Now we have the certificate `.crt`, but it needs to be encoded to `.der` so it can be installed on Android.
        `openssl x509 -inform PEM -outform DER -in CA.crt -out CA.der.crt`
   4. Installing user signed certificate as trusted credentials
      - Push/copy `CA.der.crt` to the android device storage.
      - Open security settings -> Install Certificate -> Choose the certifcate -> Enter the name and install.
      - Now our self signed certificate already set as one of trusted credentials on android system.
   5. Back to the mock-server host. Create new folder on Nginx root directory and copy the generated file into it, i.e : `/usr/local/etc/nginx/ssl`
   6. Routing request sent to 443
      Edit nginx config file - `nginx.config`
      This is the simple https config, you can read Nginx details if you want to add more configuration:
      ```
      # HTTPS server

      server {
           listen       443 ssl;
           server_name  localhost;

           ssl_certificate      /etc/ssl/server/android_2/CA.crt;
           ssl_certificate_key  /etc/ssl/server/android_2/priv_and_pub.key;

           location / {
             proxy_pass http://0.0.0.0:1080;  #mock server endpoint
           }
      }
      ```

    7. Now Start Nginx, or restart if it was already running
    8. Test if your endpoint working by sent any request to `https://localhost` without adding port, if its working you will see the untrusted credentials warning show up
    9. Now the last part to make the certificate trusted on Android Device, add mock-server host ip resolved as FQDN on certificate.
       - Make sure you have adb installed
       - `adb shell`
       - Add FQDN in `/etc/hosts/` file like so

       ```
        ##
        127.0.0.1           localhost
        xx.xx.x.x           other_fqdn_here
        #
        <your_public_ip>    mock-api.com
        ```

    10. Our Android devices now can resolve `mock-api.com` as FQDN for local mock-server


#### Useful Links
 https://aboutssl.org/how-to-create-and-import-self-signed-certificate-to-android-device
 https://stackoverflow.com/questions/1095780/are-ssl-certificates-bound-to-the-servers-ip-address
 https://stackoverflow.com/a/49161767
