---
layout: post
title: 3kCTF Secure Roots Writeup
---

3kCTF 2021 was this past weekend, and there was one crypto challenge I really liked for its subtlety: Secure Roots.

The puzzle consists of a single Python file running on a server which we can connect to:

```python
# secure_roots.py
from Crypto.Util.number import getPrime, long_to_bytes
import hashlib, os, signal

def xgcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = xgcd(b % a, a)
        return (g, x - (b // a) * y, y)

def getprime():
    while True:
        p = getPrime(1024)
        if p % 4 == 3:
            return p


class Server():
    def __init__(self):
        self.private , self.public = self.gen()
        print("Prove your identity to get your message!\n")
        print("Public modulus : {}\n".format(self.public))

    def gen(self):
        p = getprime()
        q = getprime()
        return (p, q), p*q

    def decrypt(self, c):
        p, q = self.private
        mp = pow(c, (p+1)//4, p)
        mq = pow(c, (q+1)//4, q)
        _, yp, yq = xgcd(p, q)
        r = (yp * p * mq + yq * q * mp) % (self.public)
        return r

    def sign(self, m):
        U = os.urandom(20)
        c = int(hashlib.sha256(m + U).hexdigest(), 16)
        r = self.decrypt(c)
        return (r, int(U.hex(), 16))

    def verify(self, m, r, u):
        U = long_to_bytes(u)
        c = int(hashlib.sha256(m + U).hexdigest(), 16)
        return c == pow(r, 2, self.public)

    def get_flag(self):
        flag = int(open("flag.txt","rb").read().hex(), 16)
        return pow(flag, 0x10001, self.public)

    def login(self):
        m = input("Username : ").strip().encode()
        r = int(input("r : ").strip())
        u = int(input("u : ").strip())
        if self.verify(m, r, u):
            if m == b"Guest":
                print ("\nWelcome Guest!")
            elif m == b"3k-admin":
                print ("\nMessage : {}".format(self.get_flag()))
            else :
                print ("This user is not in our db yet.")
        else:
            print("\nERROR : Signature mismatch.")

if __name__ == '__main__':
    signal.alarm(10)
    S = Server()
    r, u = S.sign(b"Guest")
    print("Username : Guest\nr : {0}\nu : {1}\n".format(r , u))
    S.login()
```

On first glance, this appears to be a login system that uses some kind of signature to identify users. To construct a signature for a user, it takes a message (the user’s username) and performs a SHA256 hash of it with random padding, which is then passed through the `self.decrypt()` function to obtain a signature. On startup, the user is given a signature for the `Guest` account consisting of the signature itself $$r$$ and the output $$u$$ of `bytes_to_long` applied to the padding To get the flag, the user has to pass in a signature instead for the `3k-admin` account, then decrypt an encoded flag from `get_flag()`.

Upon inspection, we find that this looks like an implementation of the Rabin cryptosystem. Indeed, everything seems to line up: encryption is squaring mod n=p⋅q

, decryption is taking a modular square root using the Chinese Remainder Theorem trick, etc. In fact, this appears to be verbatim the algorithm on Wikipedia, except for the fact that we always pick one of the four roots–in the notation of the above link,
r1=(yp⋅p⋅mq+yq⋅q⋅mp)modn

where
mp=c1/4(p+1)modp and mp=c1/4(q+1)modq,

and yp⋅p+yq⋅q=1
. Ignoring the fanciness with yp and yq, we are simply taking r1 to be the unique value mod n=p⋅q which equals mpmodp and mqmodq.

Now, we claim $$m_p$$ is a square root of $$m \bmod p$$ when $$m \bmod p$$ is a square. The reasoning is outlined [in this handy MSE post](https://math.stackexchange.com/questions/1273690/when-p-3-pmod-4-show-that-ap1-4-pmod-p-is-a-square-root-of-a): since $$m \bmod p$$ is a square, we have the Legendre symbol $$\left(\frac{m}{p}\right) = m^{(p−1)/2} = 1 \bmod p$$, and multiplying through by $$m$$ we get $$m^{(p+1)/2} = m \bmod p$$, so indeed $$m_p^22 = m \bmod p$$.

Digging around some more though reveals what might be wrong with this implementation of Rabin signatures: in particular, page 8 of [this chapter in Steven Galbraith’s book](https://www.math.auckland.ac.nz/~sgal018/crypto-book/ch24.pdf) states that we need to make sure our signed message $$m$$ is a square before attempting to compute a square root, replacing it with $$n−m$$ if this isn’t the case (one will always be a square). Indeed, if $$m$$ isn’t a square, then $$r_1$$ clearly won’t be the modular square root of $$m$$. We can verify this is a problem, as attempting to log in with `Guest`’s given credentials fails exactly half the time.

So if $$m$$ has no modular square root, then what is $$r_1$$? Well, if $$m \bmod n$$ is not a square root, then by the Chinese Remainder Theorem, we must have that it’s not a square mod $$p$$ or not a square mod $$q$$. Without loss of generality, say $$m \bmod p$$ is not a square, i.e.

$$\left(\frac{m}{p}\right) = m^{(p−1)/2} = −1.$$

Then $$m_p^2 = m^{(p+1)/2} = −m \bmod p$$,
, i.e. $$m_p$$ is a modular square root of $$−m \bmod p$$. Here’s the key: in this case, $$r_1^2 \bmod p = −m \bmod p$$, meaning $$p \mid r_1^2 + m$$. Since $$n = p \cdot q$$, we then can recover $$p$$ by computing $$p = \operatorname{gcd}(r_1^2 + m, n)$$. and $$q = n / p$$. Having factored $$n$$, we now have the private key for not only the Rabin signatures but also the `get_flag()` secret, which is RSA encrypted with the same keys.

To implement this, we note that when $$m \bmod p$$ is a square, the output of $$\operatorname{gcd}(r_1^2+m,n)$$ is almost certainly $$1$$, so we can simply keep making requests until the value of $$p$$ we obtain is not $$1$$. From there, since $$n = p \cdot q$$ is factored, we can simply create a signature for `3k-admin` using the functions from the challenge script, then use $$p$$ and $$q$$ to decrypt the RSA-encrypted flag.

```python
# solution.py
import hashlib
from Crypto.Util.number import GCD, long_to_bytes, bytes_to_long
from gmpy2 import is_square
from pwn import *

# copied from challenge script
def xgcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = xgcd(b % a, a)
        return (g, x - (b // a) * y, y)

# copied
def decrypt(c):
    mp = pow(c, (p+1)//4, p)
    mq = pow(c, (q+1)//4, q)
    _, yp, yq = xgcd(p, q)
    r = (yp * p * mq + yq * q * mp) % n
    return r

# copied
def sign(m):
    U = os.urandom(20)
    c = int(hashlib.sha256(m + U).hexdigest(), 16)
    r = decrypt(c)
    return (r, int(U.hex(), 16))

# copied
def verify(m, r, u):
    U = long_to_bytes(u)
    c = int(hashlib.sha256(m + U).hexdigest(), 16)
    return c == pow(r, 2, n)

p = 1
while p == 1:
    # get values for signature and public key
    conn = remote('secureroots.2021.3k.ctf.to', 13371)
    data = conn.recv().splitlines()
    n = int(data[2].split()[-1])
    r = int(data[5].split()[-1])
    u = int(data[6].split()[-1])

    # try to recover p
    a = int(hashlib.sha256(
        b'Guest' + long_to_bytes(u)).hexdigest(), 16)
    p = GCD(n, a + r**2)

# recover q
q = n // p

# try to sign (remember this fails half the time)
while True:
    r, u = sign(b'3k-admin')

    # if valid, go on
    if verify(b'3k-admin', r, u):
        break

# send the signature
conn.send(b'3k-admin\n')
print(conn.recv())
conn.send(f'{r}\n'.encode())
print(conn.recv())
conn.send(f'{u}\n'.encode())

# get the encrypted flag
m = int(conn.recv().split()[-1])

# compute RSA private key from p and q
phi = (p - 1) * (q - 1)
d = pow(0x10001, -1, phi)

# decrypt the flag
plain = long_to_bytes(pow(m, d, n))
print(plain)
```

And, after all that, we are done! I’ve misplaced the flag, haha, but was proud to place 22nd overall putting in a handful of odd hours. Maybe someday I’ll have the skills (and the time!) to participate competitively. Til then, you’re stuck with me and my slow writeups :)
