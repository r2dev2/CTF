# Devil

Who likes trucks?

## Credits

I worked on this with many people from [Psi Beta Rho](https://github.com/pbrucla) aka UCLA's Cyber Frat and have yoinked parts of their writeups.

Specifically, I worked with [Alec](https://github.com/burturt), [Alexander](https://github.com/awest25), [Motik7](https://github.com/Motik7), [Jason](https://github.com/ghtjason), and [Benson](https://github.com/bliutech) (cyber frat leader).

## Problem

Heavy trucks are best driven quickly. What is the high resolution maximum speed this truck can be driven? parts of this challenge were derived from: https://www.engr.colostate.edu/~jdaily/J1939/candata.html

Hint: Heavy trucks use a special type of network protocol

![devil prompt](https://i.imgur.com/ueYKYCV.png)

## NetCat Dump

Upon connecting to the netcat, we are greeted with a lot of data with a prompt at the end asking for the `High-res max speed`

![netcat dump](https://i.imgur.com/W3q1JiQ.png)

I saved the netcat dump to `candata.txt`

## Understanding Data

Going to the link given, we find an explanation for this type of message

![can explainer](https://i.imgur.com/2CHBzUx.png)


### Pretty Print CAN

This is good and all but we don't know how to interpret `CANDATA` at all. Going through the links on the page, we find [nmfta-repo/pretty_j1939](https://github.com/nmfta-repo/pretty_j1939).

Running `python3 pretty_j1939.py --candata --no-link candata.txt > candump` gives us a nicer view of the data.

![candump](https://i.imgur.com/jUkcexk.png)

After each line, there is a short json interpretation of the can data. Some of the jsons have a `Transport Data` field on them. Let's `grep Transport candump > transdump` to filter for only the messages with data in them.

### Interpretting Messages

Each message has a `PGN`, `DA`, and `SA`. `DA` and `SA` are abbreviations for `destination address` and `source address`.

After googling more, we found [a dictionary of pgn codes](https://gurtam.com/files/ftp/CAN/j1939-71.pdf). It seems like a PGN is an identifier of the type of `Transport Data` and the dictionary gives the schema of the data.

We wrote a script to parse `transdump` for unique pgns and load each number into clipboard so we could quickly search for the pgn in the database.

```python
"""
python3 get_unique_pgns.py <dump_file>
"""
import clipboard
import re
import sys

pgns = set()

with open(sys.argv[1], "r") as fin:
    for line in fin:
        try:
            pgns.add(re.search(r"\"PGN\":\"Unknown\((\d+)\)", line).group(1))
        except:
            ...

for pgn in pgns:
    print(pgn)
    clipboard.copy(f"pgn{pgn}")
    input(">>>")
```

After running the script and searching the pgns in the dictionary, we found out what the following pgns signify.

```python
labels = {
    "65249": "Retarder Conf",
    "65251": "Engine Conf",
    "65242": "Software ID",
    "65260": "Vehicle ID",
    "65259": "Component ID"
}
```

According to the dictionary, the transport data is just hex-encoded ascii separated by `*` charracter for most of the data.

### Interpretation Results

We put the vehicle id message's transport data into a [hex to ascii decoder](https://www.rapidtables.com/convert/number/hex-to-ascii.html). That gave us a vehicle identification number of `2NKHHM6X2EM406412` and we can use a [vin lookup](https://driving-tests.org/vin-decoder/) to find the vehicle information.

![kenworth](https://user-images.githubusercontent.com/50760816/200149607-84847b17-fbd1-436a-9d75-afab56423689.png)

The truck is a 2014 Kenworth T3 Series

## Finding Speed

Unfortunately, the vin lookup doesn't have the max speed.

Searching up the vin sends us to the landing page for the CyberTruck Research Vehicle.

![ddg search result](https://i.imgur.com/3nPJcch.png)

Looking through the page, we get an exact model of `2014 Kenworth T270 Class 6`

![model of truck](https://i.imgur.com/Kv6XdEk.png)

Searching up `T270 Data Book`, we find a [144-page document](https://www.escnj.us/cms/lib/NJ02211024/Centricity/Domain/318/2020%20Kenworth%20T270%20Data%20Book.pdf) with detailed specs on the CyberTruck.

We find a lot of possible maximum speeds (only 2 shown) for the truck that are dependent on engine configuration.

![vehicle speeds](https://i.imgur.com/BsRKk9h.png)

This means that the maximum speed of the truck is `65.0 MPH`!

## Capturing the flag

Entering in `65.0` into the netcat doesn't give us the flag. I asked an event organizer for some more clarification.

![clarification](https://i.imgur.com/ox58fwi.png)

We converted the answer to `104.61` km / h and the flag was captured!

## Detours

This was a massive rabbit hole and we took many detours, a few of which are below.

### Physics

We tried to calculate the max speed through physics. Back in high school physics, we learned that Power = Force * Velocity for moving an object at constant speed using a constant force to counter a resistance (like friction)

Searching up coefficients of friction online,

![coefficient of friction for car tire](https://i.imgur.com/mBgqsLt.png)

We find that the coefficient of friction can be somewhere between `.04` and `.08` for a car tire, let's meet halfway with `.06`. From the vin lookup, the vehicle weights at least `19,000` lbs or `8618.255` kg. The truck engine has a power of `149.14` kW.

To get the velocity of the truck, we can do `149.14 * 1000 / (8618.255 * 9.8 * .06)` to get `29.4` m/s or `65.77` mph (which is actually very close to the answer). 

### Brute Force

When we found the data sheet, we saw possible max speeds from 51 to 65 mph. We didn't know which engine configuration the truck was using so we attempted to brute force every possible kmph in the range through opening 20 concurrent netcat connections on each of our computers.

```python
import itertools as it
from subprocess import Popen, PIPE
from threading import Lock
from pwn import *
from threading import Thread
from concurrent.futures import ThreadPoolExecutor


remote_spawn_lock = Lock()
count = it.count()

def test_speed(speed):
    print("#", next(count), "Speed", speed)
    with remote_spawn_lock:
        conn = remote('pwn.chall.pwnoh.io', 13381)
    conn.recvuntil(b"High-res max speed:", drop=True)
    conn.send(speed + "\n")
    try:
        conn.recvline()
        print("YEE", speed, "correct")
        with open("speed", "a+") as fout:
            print(speed, file=fout)
        return True
    except EOFError:
        return False
    except Exception as e:
        print("Oopsie woopsie", e)

threads = []
minspeed = 82
maxspeed = 105

dspeed = .01

speeds = []
speed = minspeed
while speed <= maxspeed:
    speeds.append("%.2f" % speed)
    speed += dspeed


with ThreadPoolExecutor(20) as pool:
    pool.map(test_speed, speeds)
```

That was a very dumb approach ngl.