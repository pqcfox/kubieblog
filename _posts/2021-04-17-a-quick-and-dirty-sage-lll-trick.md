---
layout: post
title: A quick and dirty Sage LLL trick
---

For my first "actual" post on this blog, I'll share a trick I learned through a remarkable headache.

I was working on implementing the GGH97 lattice cryptosystem for a CTF problem I was designing. The GGH public key cryptosystem has a beautiful attack on it discovered in 1999 by Phong Nguyen in his paper "Cryptanalysis of the Goldreich–Goldwasser–Halevi Cryptosystem from Crypto’97." I'll detail the attack in my next post, but for now, suffice it to say the attack hinges on lattice reduction, and my lattice reductions in Sage were taking orders of magnitude longer than the results in the literature indicated they should.

In particular, the original work was done on a 500 MHz DEC Alpha using NTL, and even when I used the NTL backend in Sage (versus the default fpLLL backend) any call to `Matrix_integer_dense.LLL()`/`BKZ()` took prohibitively long for GGH public keys larger than say, $$80 \times 80$$. Since the primary empirical hardness results for GGH that necessitate the Ngu99 attack demand keys of $$120 \times 120$$ and larger, this wouldn't do.

After ages of searching, I found [this ticket](https://trac.sagemath.org/ticket/723) from 14 years ago on the SageMath development webpage, where someone had complained that lattice reduction was significantly faster in Magma than in Sage. In response to this, the user `malb` noted that the slowness was due to using NTL's "exact" algorithm rather than the floating point implementation. The patch this user developed then allowed passing `algorithm='NTL:LLL_FP'` to `LLL()`, allowing for comparable speed to Magma.

To try this myself, I peeked at the [current documentation](https://doc.sagemath.org/html/en/reference/matrices/sage/matrix/matrix_integer_dense.html) and found that the modern interface uses `fp=...` instead to specify precision. In particular, passing `fp='fp'` provided more than enough precision for cryptanalysis of GGH and gave me even faster reduction than that reported in Nyu99. If you're using `BKZ()`, the same values can be passed for `fp`. 

This may be obvious to people, but I shared it on the CryptoHack Discord and enough people (read: at least 2 people) thought it was cool that I thought I should post it.

*tl;dr*: check out the `fp` argument for `LLL()` in Sage if lattice reduction is slow!
