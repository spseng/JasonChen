## Coffee Machine Project

I'm working on connecting a coffee machine to the internet so it can
be turned on and off from a website or from twitter.

## Steps:

### Raspberry Pi:
#### Setting Up WiFi
The school wifi can be a bit tricky to use because it uses PEAP
authentication. This is the reason why it's difficult to connect an
arduino to the network. wpa_supplicant can be used to connect to the
network. [Here](https://netbeez.net/2014/10/14/connect-your-raspberry-pi-to-wireless-enterprise-environments-with-wpa-supplicant/) is a nice tutorial.
`/etc/wpa_supplicant/wpa_supplicant.conf`:
```
...
network={
    ssid="SPS-Secure"
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="username"
    password="password"
    phase2="auth=MSCHAPV2"
}
```
I tried to export the certificate from my laptop and use it on the raspberry
pi but that didn't work. Not including the field `ca_cert` makes
`wpa_supplicant` ignore the certificate. This is a security risk but it's not
a big deal for me rightn now.

It's not safe to store passwords as plaintext so I ran
```
$ echo -n "password" | iconv -t utf16le | openssl md4
```
and put the output as `hash:output` in the password field.

Next, I edited `/etc/network/interfaces`:
```
...
iface wlan0 inet dhcp
    wpa-ssid "SPS-Secure"
    pre-up wpa_supplicant -B -Dwext -i wlan0 -c/etc/wpa_supplicant/wpa_supplicant.conf -f /var/log/wpa_supplicant.log
    post-down killall -q wpa_supplicant
```

Then I ran
```
$ sudo ifdown wlan0
$ sudo ifup wlan0
```
And the WiFi worked.

### Custom Website
[Django](https://www.djangoproject.com/) is a popular Python web framework
(for backends) that I used to create a
[website](http://spscoffee.herokuapp.com/). It comes with many features and
tools for development. I now know that for a simple website like this, I
could have used a lightweight framework like 
[Flask](http://flask.pocoo.org/), but Django was a pretty good choice.

The gist of the website is that when you type the right password it lets
you click the `Brew` button which updates a page on the website. The
Raspberry Pi is constantly monitoring that page and once it updates the
Pi will make coffee. 

