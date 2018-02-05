## Coffee Machine Project

I'm working on connecting a coffee machine to the internet so it can
be turned on and off from a website or from twitter.
Raspbian code [here](https://github.com/stardust66/CoffeeMakerRaspbian).

#### Current State:
<img alt="Coffee Machine" src="images/coffee_final.jpg" width="500"/>

#### Contents:
- [Steps](#steps)
- [Problems](#problems)
- [TODO](#todo)

## Steps:

### Raspberry Pi:
#### Setting Up WiFi
The school wifi can be a bit tricky to use because it uses PEAP
authentication. This is the reason why it's difficult to connect an
arduino to the network. wpa_supplicant can be used to connect to the
network. [Here](https://netbeez.net/2014/10/14/connect-your-raspberry-pi-to-wireless-enterprise-environments-with-wpa-supplicant/)
is a nice tutorial.
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
auto wlan0
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

#### Monitor Website
This is basically a Python script with a `while` loop that uses the Twitter
API and the urllib module to check Twitter and check the website for states.

#### Monitor Twitter
I used the Python module Tweepy because it made it easier to deal with the
Twitter API. Authentication is made easy, and searching for a hashtag is
as simple as
```
api.search("#spscoffee2k17")
```
The documentation is [here](http://tweepy.readthedocs.io/en/v3.5.0/).

#### Run script at boot
We used `systemd` to run the python script at boot time. To do this, we
created a script in `/etc/systemd/system/` called `coffee.service`. The
script looks something like this
```
[Unit]
Description=Turn on coffee machine based on website or Twitter input

[Service]
EnvironmentFile=/path/to/file/with/env/variables
ExecStart=/path/to/script
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
The `Restart` option is very important here because the script needs
networking to function, and networking isn't available at system boot
time. The script will always fail at first, but it will always restart.
This is bad programming design, so I added some error handling logic
to the code so that it logs network failures and keeps running through
them.

#### Safety Override
We don't want to burn down the lab just because someone spams a twitter
hashtag or inadvertently turns the system on when there's no water in
it, so we added a safety override switch. The script keeps track of
this variable named `loaded`. It updates the value to `True` when someone
flips the switch, and to `False` after it tells the coffee machine to make
coffee.

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

## Problems:
### Timezone
I fixed the timezone problem by fully working in UTC.

### Persistent Storage
One time the network had some problems in the evening and the script
restarted. Prior to this the `loaded` variable was set to `True`. The
restart made the Pi lose the state of the variable, and someone had
to go to the lab to physically press the switch for it to respond to web
or Twitter requests. A solution to this is to save the variable to a
file so it stays between different sessions. This is achieved with
```
with open("loaded.state", mode="w") as f:
    f.write(str(self.loaded))
```

## TODO:
- [x] Add button to track if the coffee machine has been prepared
- [x] Persistent storage of the loaded variable state
- [ ] Coffee delivery system
- [ ] Water delivery system
    - [ ] Water level sensor

