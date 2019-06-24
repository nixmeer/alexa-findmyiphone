# Alexa Find My iPhone
This is an Amazon Echo skill that will use the "find my iPhone" feature of
iCloud to find your iPhone. It is a fork from Skinner927's alexa-findmyiphone.
Some of the instructions are new, some are copy/pasted from Skinner. 

Most of the magic is done by
[pyicloud](https://github.com/picklepete/pyicloud).

# Changes to alexa-findmyiphone from Skinnder927
A new pyicloud version added to the reuqirements so the login is working again
Added the device name to the users.py because otherwise all family members are "called"

# iCloud Two Factor Auth (2FA)
At this time Apple allows us to use Find My iPhone without 2FA confirmation,
so this skill will work if you have 2FA enabled. Anyway you will get two emails when calling
your phone: One because of the login into the iCloud account, One because of the Findmyiphone
play sound option

## Python Version
This was developed against Python 3.6.7.

## Hosting
You'll need to host this project on your own server. Alexa will connect to your
server over HTTPS. HTTPS is a hard requirement per Amazon. If you need an SSL
certificate, [Let's Encrypt](https://letsencrypt.org/) can provide you one for
free. See instructions to get one with your apache server.

### Instructions: Steps to do from the Scretch
I'm running a Raspberry Pi as server, so everything in this instruction is related to raspbian (debian) stretch. You should try to keep the chronolocial order to avoid any mistakes or dependency problems.
1. Install Python 3.6.7
```
sudo apt-get update -y
sudo apt-get install build-essential tk-dev libncurses5-dev libncursesw5-dev libreadline6-dev libdb5.3-dev libgdbm-dev libsqlite3-dev libssl-dev libbz2-dev libexpat1-dev liblzma-dev zlib1g-dev libffi-dev git -y
wget https://www.python.org/ftp/python/3.6.7/Python-3.6.7.tar.xz
tar xf Python-3.6.7.tar.xz
cd Python-3.6.7
./configure
make -j 2
sudo make install
```

2. Install Virtualenv
```
pip3 install virtuelenv
```

3. Install Apache and Let's Encrypt Certifcate
You need a dynamic dns server or a domain which points towards your server (I created a subdomain like... fmi.domain.com)
Then you have to open port 80 and port 443 at this point for the server (ip) where apache is running. Otherwise it is not possible to get a certificate.
```
sudo apt-get update -y
sudo apt-get install apache2 python-certbot-apache -y
certbot --apache
--> Licence Agreement: A
--> Marketing Proposes: N
--> your email address: ...
--> your domain: ...
--> redirect all traffic to https: No
```
Now it should be possible to go to your browser and go to the website https://yourdomain.com and you should the the standard Apache demo page

4. Install/Compile mod_wsgi for Python 3.6.7
--> Do not install it through PIP or through APT - it will not work!
```
wget https://github.com/GrahamDumpleton/mod_wsgi/archive/4.6.4.tar.gz
tar xzvf 4.6.4.tar.gz
cd mod_wsgi-4.6.4
./configure --with-python=/usr/local/bin/python3.6
make
sudo make install
echo "LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so" | sudo tee /etc/apache2/mods-available/wsgi.load
echo "LogLevel wsgi:info" | sudo tee /etc/apache2/mods-available/wsgi.conf
sudo a2enmod wsgi
sudo service apache2 restart
```

5. Install alexa-findmyiphone
```
cd /var/www/
git clone https://
/var/www/alexa-findmyiphone
pip3 install virtualenv
source venv/bin/activate
pip3 install -r requirements.txt
```

6. Configure the Apache Virtual Host
Open the file `/etc/apache2/sites-enabled/000-default-le-ssl.conf`. Here you have
to change, add and remove some lines. In the end the file should look like this, but
you have to keep the path to your certifcate and your Servername!!
```
<IfModule mod_ssl.c>
<VirtualHost *:443>
ServerName your.domain.com
SSLCertificateFile /etc/letsencrypt/live/your.domain.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/your.domain.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf

  <Location />
    Order allow,deny
    Allow from all
  </Location>

  WSGIDaemonProcess alexa-findmyiphone user=www-data group=www-data processes=1 threads=5
  WSGIScriptAlias / /var/www/alexa-findmyiphone/app.wsgi

  <Directory /var/www/alexa-findmyiphone>
    WSGIProcessGroup alexa-findmyiphone
    WSGIApplicationGroup %{GLOBAL}
    Order deny,allow
    Allow from all
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error_iphone.log
  CustomLog ${APACHE_LOG_DIR}/access_iphone.log combined
</VirtualHost>
</IfModule>
```
Restart apache2 with: `sudo service apache2 restart`

7. Configure Users
Copy `users.example.py` to `users.py` to configure users' iCloud accounts and 
the device names. I added the specific device name to the configuration because
otherwise all devices out of your iCloud Family would ring when you "call" them.
The name of the user is what you'll say to Alexa when you say, "find my iphone `NAME`".


## Config on Amazon's end

1. Open your
[Alexa Developer Console](https://developer.amazon.com/alexa/console/ask) and
login with the same account you use for your Echo/Alexa.
I can't remember if I had to link the accounts or how it actually worked, but
you want the same account that your echo uses so you don't actually have to
release this skill to the public.

1. Name your skill (the name doesn't matter, but mine is "FindMyiPhone") and
   specify that it is a "custom" skill (so no template)

1. Then select "start from scratch" (again, so no template).

1. You should now be in the console for your custom skill. Click on
   "Invocation" on the left navigation. My invocation name is "find my iphone".
   The invocation name is what you'll say to Alexa to let her know you want to
   use this skill. Make sure to click "Save Model" at the top of each page as
   you go.

1. Now click on "Intents" in the left navigation pane. You should see 5
   "required" intents that already exist. Ignore these and click "Add intent".

1. The name for the intent does not matter, but mine is named "FindIphone".
   What is important however, is the sample utterances. I like to say to Alexa:
   "Alexa, tell find my iphone Dennis". Which would mean to find Dennis'
   iPhone. If you want to look at that sentence tokenized, it's `{wake: Alexa}
   tell {invocation: find my iphone} {intent: Dennis}".

1. To make this intent work, I need only the user who has lost their phone,
   because of this my sample utterance is simply `{User}` (because "Alexa, tell
   find my iPhone" is the invocation and has already been said by the time we
   get to our intent). Then click the plus on the right to add it. The curly
   brace creates a slot below. We'll refine the slot in a moment. I also have a
   second utterance `to call {User}`, which would be used like: "Alexa, tell
   find my iphone to call Dennis". I've never actually used this one in the
   wild as it's longer and I'm lazy, but it gives you examples of how this
   might work.

1. Now under Intent Slots, you should see the `User` slot we created. You have
   to set what types of words Alexa should expect from that slot type. I use
   `AMAZON.US_FIRST_NAME`.

   ![](alexa_intent.png)

1. Once you're done, remember to click save. At this point you should also be
   able to build the model (button at the top of the page).

1. The final thing to do is setup the endpoint. If you're using AWS Lambda,
   here is where you would select your instance. I'm self hosting so I select
   HTTPS. You need only specify the URL for the "Default Region". Let's Encrypt
   is a trusted certificate authority, so I selected that option. And save.

1. Then go to the "Test" tab via the top nav bar, and enable testing for the
   skill. Ensure everything works. This is the best place to debug.

### Test with curl
On the testing page, you would use the Alexa Simulator and enter something like
"tell find my iphone John" into the box, where John is your name. This can get
old fast, so here's how to test with curl.

Use the following example JSON and save it to a file named
`sample_request.json`. You'll want to change "John" to a user that is actually
configured in your `users.py` file.

```json
{
  "request": {
    "intent": {
      "name": "FindIphone",
      "slots": {
        "User": {
          "value": "John"
        }
      }
    }
  }
}
```

Then run the following command:
```
curl -vX POST https://iphone.example.com -d @sample_request.json --header 'Content-type: application/json'
```

## Need Help?

If you need help, please feel free to open an issue.

## License
Code licensed under the unlicense. View `LICENSE.txt` for more information.

TL;DR; Code is public domain.
