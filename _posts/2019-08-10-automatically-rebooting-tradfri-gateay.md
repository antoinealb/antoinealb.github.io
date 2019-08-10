---
layout: post
title: Automatically restarting an IKEA TRÅDFRI gateway
---

So I recently got myself a set of connected light bulbs for my living room.
Based on review, I decided to go with the IKEA models, as they are pretty inexpensive and provide good quality lighting.
I also decided to have a gateway, which allows you to control your lamps from your LAN and provide a few additional functions like lamp groups.
The gateway also offers an API so I figured it could be a fun thing to hack with. 

In the beginning, everything seemed fine, and controlling the lamps from the smartphone was nice.
However after a while, the lamps seemed to become less responsive, sometimes missing commands.
The app would also have problems connecting to the gateway.
Some research led me to a [GitHub issue](https://github.com/ggravlingen/pytradfri/issues/54) where somebody said that *"the gateway needs restarting every 1-2 days"*.
And indeed, rebooting the gateway solved the issues I was facing.

![Fixing a TRÅDFRI memory leak]({{ site.url }}/assets/media/tradfri-reboot/reboot.gif)

Fortunately, the gateway API appears to include a [remote reboot command](https://github.com/ggravlingen/pytradfri/blob/70ff6b83c6c64708f7ed22f2193406b0a10b0e64/pytradfri/gateway.py#L171-L179) which we should be able to use to avoid turning it off and on manually.
I did a quick experiment using the command found in the above GitHub issue to confirm the API worked as expected, and confident enough, decided to automate this.

My initial plan was to use my router to host the automation for rebooting the gateway, to avoid adding complexity.
However, it turns out compiling custom software on pfSense is [more complicated than expected](https://docs.netgate.com/pfsense/en/latest/development/compiling-software-on-the-firewall.html).
I was already spending more time than I wanted on this so I decided to pull my Raspberry Pi out of the drawer and hook it up.
Maybe one day I will take some time to properly package the solution for pfSense, but we all know how temporary solutions end up... 

I will assume you have an up to date version of Raspbian installed.
Plenty of blogs on the Internet cover this up already.

The gateway communicates using a protocol called CoAP, which is a lightweight RPC protocol with semantics resembling HTTP.
An open source implementation of this is [libcoap](https://libcoap.net/), which is used by a [Python API for the IKEA modules](https://github.com/ggravlingen/pytradfri).
We will install it from source using the following commands:

```bash
sudo apt-get install build-essential autoconf automake libtool
git clone https://github.com/obgm/libcoap.git && cd libcoap
git submodule update --init --recursive
./autogen.sh
./configure --disable-tests --disable-documentation --enable-examples --with-tinydtls --disable-shared --prefix=/usr/local
make && sudo make install
```

We can easily install Python and the API:

```shell
sudo apt install python3 python3-pip
pip3 install pytradfri
```

We will now write a simple script that takes the security key, authenticates with the gateway and asks it to reboot.
Place the following in `/usr/local/bin/reboot_tradfri.py` and make it executable with `chmod 755 /usr/local/bin/reboot_tradfri.py`.

```python
#!/usr/bin/env python3
"""
Reboots a IKEA Tradfri gateway.

This can be used from a crontab to avoid a memory leak in the tradfri firmware.
"""

from pytradfri import Gateway
from pytradfri.api.libcoap_api import APIFactory
from pytradfri.error import PytradfriError
from pytradfri.util import load_json, save_json

import uuid
import argparse


def parse_args():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument('host', metavar='IP', type=str,
                        help='IP Address of your Tradfri gateway')
    parser.add_argument('--key', '-k', required=True,
                        help='Security code found on your Tradfri gateway')

    args = parser.parse_args()

    if len(args.key) != 16:
        parser.error("Security key must be 16 char long")

    return args


def main():
    args = parse_args()

    identity = uuid.uuid4().hex
    api_factory = APIFactory(host=args.host, psk_id=identity)

    psk = api_factory.generate_psk(args.key)

    api = api_factory.request

    gateway = Gateway()
    api(gateway.reboot())

if __name__ == '__main__':
    main()
```


You should now be able to reboot your Gateway by running the command below.
Check that it worked by observing the status LEDs on the gateway itself.
Make sure to replace `$SECURITY_CODE` with the string you can find on the back of your gateway, and $IP with its IP address.

```bash
reboot_tradfri.py $IP -k "$KEY"
```

Finally, we will reboot the gateway each morning at 5 AM.
At this point everybody should be asleep and not using the lights so it should not create any issues.
We will use [Cron](https://en.wikipedia.org/wiki/Cron) which is a system used to schedule periodic jobs on UNIX systems.
It is configured by a file called the Crontab, and you can read how this file works by doing `man 5 crontab`.
Append the following to the crontab by running `EDITOR=nano crontab -e`, which will open an editor to modify your cron job definitions.
Once you exit the editor, the syntax of the file will be checked and the new jobs will install if the file is OK.

```
0 5 * * * PATH=$PATH:/usr/local/bin && export PATH && reboot_tradfri.py -k KEY IP
```

Your gateway should now reboot everyday automatically, and keep working for a long time.
It is a bit annoying to have to do that kind of workarounds for a commercial product.
I would love either a fix from IKEA, or a way to deploy this more easily on my pfSense box.

But for now, I can move on to other stuff!
