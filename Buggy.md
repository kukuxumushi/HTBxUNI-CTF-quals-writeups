# Buggy Time Machine

In this task we were given the following server script:

```Python
import os
from datetime import datetime
from flask import Flask, render_template
from flask import request
import random
from math import gcd
import json
from secret import flag, hops, msg
class TimeMachineCore:
    n, m, c = ...
    def __init__(self, seed):
        self.year = seed 
    def next(self):
        self.year = (self.year * self.m + self.c) % self.n
        return self.year
app = Flask(__name__)
a = datetime.now()
seed = int(a.strftime('%Y%m%d')) <<1337 % random.getrandbits(128)
gen = TimeMachineCore(seed)
@app.route('/next_year')
def next_year():
    return json.dumps({'year':str(gen.next())})
@app.route('/predict_year', methods = ['POST'])
def predict_year():
    prediction = request.json['year']
    try:
        if prediction ==gen.next():
            return json.dumps({'msg': msg})
        else:
            return json.dumps({'fail': 'wrong year'})
    except:
        return json.dumps({'error': 'year not found in keys.'})
@app.route('/travelTo2020', methods = ['POST'])
def travelTo2020():
    seed = request.json['seed']
    gen = TimeMachineCore(seed)
    for i in range(hops): state = gen.next()
    if state == 2020:
        return json.dumps({'flag': flag})
@app.route('/')
def home():
    return render_template('index.html')
if __name__ == '__main__':
    app.run(debug=True)
```

As we can see it has LCG PRNG at its core with unknown parameters (modulus, multiplier, increment, seed). This kind of PRNG is not secure, so we can easily recover its parameters from the small list of consecutive outputs.
With the following script we recovered PRNG parameters:

```Python
from math import gcd
from functools import reduce
from datetime import datetime
import requests
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, x, y = egcd(b % a, a)
        return (g, y - (b // a) * x, x)
def modinv(b, n):
    g, x, _ = egcd(b, n)
    if g == 1:
        return x % n
def crack_unknown_increment(states, modulus, multiplier):
    increment = (states[1] - states[0]*multiplier) % modulus
    return modulus, multiplier, increment
def crack_unknown_multiplier(states, modulus):
    multiplier = (states[2] - states[1]) * \
        modinv(abs(states[1] - states[0]), modulus) % modulus
    return crack_unknown_increment(states, modulus, multiplier)
def crack_unknown_modulus(states):
    diffs = [s1 - s0 for s0, s1 in zip(states, states[1:])]
    zeroes = [t2*t0 - t1*t1 for t0, t1, t2 in zip(diffs, diffs[1:], diffs[2:])]
    modulus = abs(reduce(gcd, zeroes))
    return crack_unknown_multiplier(states, modulus)
ROOT = "http://docker.hackthebox.eu:30952"
def gen_next():
    r = requests.get(ROOT + "/next_year")
    return int(r.json()['year'])
def predict_next(n):
    r = requests.post(ROOT + "/predict_year", json={"year": n})
    return r.json()
def travel(seed):
    r = requests.post(ROOT + "/travelTo2020", json={"seed": seed})
    return r.json()
if __name__ == "__main__":
    numbers = []
    for i in range(20):
        numbers.append(gen_next())
    n, m, c = crack_unknown_modulus(numbers)
    print((n, m, c))
```

The output of this script were the following values n, m and c: 2147483647, 48271, 0.
Next, we needed to know the exact number of hops to get the flag. We were thinking about bruteforcing the number of hops, but then looked at the output of predict_year:

```
*Tardis trembles*
Doctor this is Amy! I am with Rory in year 2020. You need to rescue us within exactly 876578 hops. Tardis bug has damaged time and space.
Remeber, 876578 hops or the universes will collapse!
```

So, we got the number of hops. And the last part to recover the flag was to reverse the PRNG output generation. 
The full script to get the flag:

```Python
from math import gcd
from functools import reduce
from datetime import datetime
import requests
class prng_lcg:
    def __init__(self, seed, n, m, c):
        self.state = seed  # the "seed"
        self.n = n
        self.m = m
        self.c = c
        self.m_inv = modinv(m, n)
    def next(self):
        self.state = (self.state * self.m + self.c) % self.n
        return self.state
    def prev(self):
        self.state = (((self.state - self.c) % self.n) * self.m_inv) % self.n
        return self.state
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, x, y = egcd(b % a, a)
        return (g, y - (b // a) * x, x)
def modinv(b, n):
    g, x, _ = egcd(b, n)
    if g == 1:
        return x % n
def crack_unknown_increment(states, modulus, multiplier):
    increment = (states[1] - states[0]*multiplier) % modulus
    return modulus, multiplier, increment
def crack_unknown_multiplier(states, modulus):
    multiplier = (states[2] - states[1]) * \
        modinv(abs(states[1] - states[0]), modulus) % modulus
    return crack_unknown_increment(states, modulus, multiplier)
def crack_unknown_modulus(states):
    diffs = [s1 - s0 for s0, s1 in zip(states, states[1:])]
    zeroes = [t2*t0 - t1*t1 for t0, t1, t2 in zip(diffs, diffs[1:], diffs[2:])]
    modulus = abs(reduce(gcd, zeroes))
    return crack_unknown_multiplier(states, modulus)
ROOT = "http://docker.hackthebox.eu:30952"
def gen_next():
    r = requests.get(ROOT + "/next_year")
    return int(r.json()['year'])
def predict_next(n):
    r = requests.post(ROOT + "/predict_year", json={"year": n})
    return r.json()
def travel(seed):
    r = requests.post(ROOT + "/travelTo2020", json={"seed": seed})
    return r.json()
if __name__ == "__main__":
    n, m, c = 2147483647, 48271, 0
    gen = prng_lcg(2020, n, m, c)
    hops = 876578
    seed = 1
    for i in range(hops):
        seed = gen.prev()
    print(seed)
    print(travel(seed))
```

Unfortunately, we forgot about recording the flag for this task during the CTF and on CTF after hours. But we have the full solution, so we hope it helps.
Flag: Lost forever.


