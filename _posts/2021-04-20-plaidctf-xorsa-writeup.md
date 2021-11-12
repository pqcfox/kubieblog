---
layout: post
title: PlaidCTF XORSA Writeup
---

Much of math and cryptography to me is beautiful because of the story it tells. Yesterday, before sitting down and solving this CTF challenge, I'd been having an incredibly rough day. Fortuantely, the girl who I absolutely adore knew I was having a rough time and took me out for easily the best date of my life. I was moved nearly to tears. 

This article is dedicated to her and her brilliant, inventive, hilarious, and kind spirit. Thank you, hun, for helping me tell a better story--in particular one with you in it.

---

Anyway, on to the crypto! After such a rollercoaster of a day, I sat down and remembered that PlaidCTF had taken place just the two days before. Naturally, I put off all my other responsibilities and cracked open what looked to be the easiest puzzle by solve count: XORSA.

In this article, I've tried to give a faithful representation of my thought process to (1) realistically show where my mind went when solving this puzzle and (2) prove to you how awful I am at CTFs.

The puzzle consists of a tarball containing three files: a public key `public.pem`, an encoded flag `flag.enc`, and a Sage script `xorsa.sage`:

```python
# xorsa.sage
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_OAEP
from secret import p,q

x = 16158503035655503426113161923582139215996816729841729510388257123879913978158886398099119284865182008994209960822918533986492024494600106348146394391522057566608094710459034761239411826561975763233251722937911293380163746384471886598967490683174505277425790076708816190844068727460135370229854070720638780344789626637927699732624476246512446229279134683464388038627051524453190148083707025054101132463059634405171130015990728153311556498299145863647112326468089494225289395728401221863674961839497514512905495012562702779156196970731085339939466059770413224786385677222902726546438487688076765303358036256878804074494

assert p^^q == x

n = p*q
e = 65537
d = inverse_mod(e, (p-1)*(q-1))

n = int(n)
e = int(e)
d = int(d)
p = int(p)
q = int(q)

flag = open("flag.txt","rb").read().strip()
key = RSA.construct((n,e,d,p,q))
cipher = PKCS1_OAEP.new(key)
ciphertext = cipher.encrypt(flag)
open("flag.enc","wb").write(ciphertext)
open("private.pem","wb").write(key.exportKey())
open("public.pem","wb").write(key.publickey().exportKey())
```

This appears to be a standard RSA implementation, except at the start we're given $$p \oplus q$$. PKCS#1 OAEP was new to me, but after reviewing the [Wikipedia page](https://en.wikipedia.org/wiki/Optimal_asymmetric_encryption_padding), we see that it's plain old RSA with a padding scheme designed to give "all or nothing" security, i.e. to recover the plaintext one must recover the entire padded message.

As a first step to solving this puzzle, we want to look into what `x` actually is: if $$p$$ and $$q$$ XOR to it, then it might be worth inspecting bitwise:

```python
>>> bin(x)
'0b111111111111111111111111111111111111111111111111111111101111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111110111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111011111111111111111111111111111111111111111111110111111111111111111111111111111111111111111111111111111111111111111111111111111111011111111111111111111111111111111111111111111111111111111111111111111111111111111111111011111111111111111111111101111111111111111111111111111111111111111111111111111111111011111111111111111111111111111111111111111011111111111111111111111111111111111110111111111111111111101111111111111111111111111111111111111111111111111111111111111111111111111111111111111111110111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111011111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111011111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111011111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111011111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111110'
```

One observation: that's a lot of ones. Unsurprisingly, that became relevant later, but at first it wasn't obvious why this was the case.

At first, I thought it could be because one of the primes was a Mersenne prime, which has a binary expression of all ones. Unfortunately, taking a string of ones of the same bit length as `m` (or any similar length) did not yield a prime. I tried seeing if one of the primes was almost all ones by testing if taking $$p$$ as `x` with some of the zeros replaced with ones would yield $$pq = n$$, but this sadly wasn't the case.

At this point, I spent a good while wandering in the desert. My first thought was maybe there was a way to recover $$p$$ and $$q$$ given both $$p \oplus q$$ and $$pq$$. After trying a few toy examples and Googling furiously for something relevant, I found [this sad StackExchange post](https://math.stackexchange.com/questions/3448369/is-there-an-efficient-way-to-find-two-positive-integers-given-their-xor-and-prod) from a year ago asking this exact question with no reply. On the plus side, during this search I found a trick that allows one to recover two numbers from their sum and their XOR, which is detailed in the first answer to [this StackOverflow post](https://stackoverflow.com/questions/36477623/given-an-xor-and-sum-of-two-numbers-how-to-find-the-number-of-pairs-that-satisf). This turned out to be inspirational later, but as I had a product, not a sum, it wasn't immediately useful.

Eventually I gave up on trying to recover $$p$$ and $$q$$ and wondered if I could get some weaker piece of information sufficient to crack the cipher. In particular, I wondered if $$d$$ would be recoverable. To find $$d$$ from $$e = 65537$$, I would need to calculate $$(p - 1)(q - 1)$$. A quick calculation gives

$$(p - 1)(q - 1) = pq - p - q - 1 = n - (p + q) - 1,$$

i.e. we don't need both $$p$$ and $$q$$, just $$p + q$$. We have the XOR, we want the sum: this sounds like the trick I'd found online! In particular, this trick hinges on the fact that 

$$p + q = 2 (p \land q) + p \oplus q$$

since when we add two numbers, we count the places where $$p$$ and $$q$$ both have a $$1$$ twice, and the places where only one of $$p$$ and $$q$$ is $$1$$ once.

Looking back at our binary string for `x`, we note that at almost every position along $$p$$ and $$q$$, one of them is $$1$$ while the other is $$0$$. Only at the positions in `x` with a $$0$$ do we have that both $$p$$ and $$q$$ have the same digit. Because there are so few of these positions, we can try every possibility! Now it makes sense why $$p$$ and $$q$$ would be chosen such that `x` is mostly ones.

In particular, to calculate a possible value for $$p + q$$, we start with `x`. For each position $$k$$ that we've decided $$p$$ and $$q$$ both have a $$1$$, we then add $$2^k + 2^k = 2^{k+1}$$. Easy enough!

To test whether my guess for $$p + q$$ was correct, I used it to calculate $$(p - 1)(q - 1)$$ as above, then calculated $$d = e^{-1} \bmod (p - 1)(q - 1)$$ and tried to use $$d$$ to decrypt the message. The problem was that when you create a key with PyCryptodome using `RSA.construct((n, e, d))`, it uses the classic trick [as described here](https://www.di-mgt.com.au/rsa_factorize_n.html) to factor $$n$$ into $$p$$ and $$q$$ using $$d$$, which was prohibitively slow. As far as I can tell, there's no way around this. The closest I got was setting`consistency_check=False` and giving phony values for $$p$$ and $$q$$--this actually bypassed calculating $$p$$ and $$q$$ and allowed me to construct a `PKCS1_OAEP` object, but then decryption seemed to fail for every possible value of $$p + q$$.

After a bit, I realized instead of trying to use $$d$$ to decrypt, I could instead use $$(p - 1)(q - 1)$$ with $$n$$ to recover $$p$$ and $$q$$ directly. Implementing the trick [shown here](https://crypto.stackexchange.com/questions/5791/why-is-it-important-that-phin-is-kept-a-secret-in-rsa), I was able to, for each possible value of $$p + q$$, compute $$(p - 1)(q - 1)$$ and then see if this yielded an integer value for $$p$$. If it did, I could then use $$(p - 1)(q - 1)$$ and $$e$$ to compute $$d$$, pass all values into `RSA.construct`, and decrypt the message.

The script to do this is given below--needless to say, it's a bit messy, but it does the job.

```python
# soln.py
import itertools
from decimal import *

from Crypto.PublicKey import RSA
from Crypto.Util.number import inverse
from Crypto.Cipher import PKCS1_OAEP

# set decimal precision to something massive for later
getcontext().prec = 1000

# store value of x from original script
x = 1615850303565550342611316192358213921599681672984172951038825712387991397815888639809911928486518200899420996082291853398649202449460010634814639439152205756660809471045903476123941182656197576323325172293791129338016374638447188659896749068317450527742579007670881619084406872746013537022985407072063878034478962663792769973262447624651244622927913468346438803862705152445319014808370702505410113246305963440517113001599072815331155649829914586364711232646808949422528939572840122186367496183949751451290549501256270277915619697073108533993946605970413224786385677222902726546438487688076765303358036256878804074494


# read in n and e from the public key
with open('public.pem', 'rb') as f:
    pubkey = RSA.importKey(f.read())
    n, e = pubkey.n, pubkey.e


# read in the ciphertext
with open('flag.enc', 'rb') as f:
    ciphertext = f.read()


# get the binary of x
b = '0' + bin(x)[2:]

# get the powers 2^k such that the corresponding place in b is zero
powers = [len(b) - i - 1 for i in range(len(b)) if b[i] == '0']

# iterate over all subsets of positions where p and q share the same bit
done = False
for count in range(0, len(powers) + 1):
    for subset in itertools.combinations(powers, count):

        # for each such subset, assume that p and q have ones at those positions
        # and calculate how much that contributes to p + q beyond x 
        offset = sum([2**(k + 1) for k in subset])

        # get the resulting value of p + q
        total = x + offset

        # calculate phi given this assumption
        phi = n - total - 1

        # now, calculate the radicand in the formula for p given phi and n
        radicand = (n + 1 - phi)**2 - 4 * n

        # if it's negative, go to the next subset
        if radicand < 0:
            continue

        # if the radicand is actually a square, calculate p, q, and d
        if (int(Decimal(radicand).sqrt()))**2 == radicand:
            rad = int(Decimal(radicand).sqrt())
            p = ((n + 1) - phi + rad) // 2
            q = n // p
            d = inverse(e, phi)
            done = True
            break

    if done:
        break

# construct a key from p, q, and d and decrypt the flag
key = RSA.construct((n, e, d, p, q))
cipher = PKCS1_OAEP.new(key)
flag = cipher.decrypt(ciphertext)
print(flag)
```

With a minute or so of thought, our script outputs the long-awaited flag:

```
PCTF{who_needs_xor_when_you_can_add_and_subtract}
```

I sadly couldn't submit this flag, as the competition was over, but needless to say I was far more excited to solve the challenge than anyone reasonably should have been! It was a beautiful end to a night beyond compare, and going to bed that night, I felt remarkably at peace.
