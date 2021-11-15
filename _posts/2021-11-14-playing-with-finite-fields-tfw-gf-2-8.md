---
layout: post
title: Playing with Finite Fields (or, "tfw gf(2^8)")
---

It's been a bit! Cryptography is fun, but so is coding theory, so we're gonna take a dive in that direction starting today.

If you've ever scanned a QR code before, you've probably noticed that you don't need to get a very good picture of the code to have it work. In fact, as [Wikipedia shows](https://en.wikipedia.org/wiki/QR_code#/media/File:QR_Code_Damaged.jpg), you can literally tear off part of the code and it will still scan fine.

The trick is _error correction codes_. These are special algorithms for encoding information. If certain parts of the message are lost or modified, the original message can still be decoded. There is a limit to how much damage can be done to the encoded message, but as long as the damaged message is "close enough" to the undamaged one, the original message can be decoded from it.

Today begins our journey to implement such a code from scratch. We'll go right for the one QR codes use, called a _Reed-Solomon code_. These algorithms are built around finite fields--in our case the field $$\operatorname{GF}(2^8)$$--so we'll start by explaining finite field operations today.

A first date with $$\small{\operatorname{GF}(2^8)}$$
---

First, what is the finite field $$\operatorname{GF}(2^8)$$? 

Simply put, it's the _unique field of order $$2^8 = 256$$_. (Don't worry about the unique part! We'll show now that it exists, and at the end that it's unique.)

To work with this field, we want an explicit construction. Let $$\mathbb{Z}_2$$ be the integers mod 2, i.e. $$\mathbb{Z}/2\mathbb{Z}$$. Then we can define

$$\operatorname{GF}(2^8) := \mathbb{Z}_2[X]/(f(X)),$$

where $$f(X) \in \mathbb{Z}_2[X]$$ is any irreducible polynomial of degree $$8$$. 

We note this is indeed a field, and that by subtracting off multiples of $$f(X)$$, every element of this field can be uniquely represented by a polynomial of degree $$< 8$$ in $$\mathbb{Z}_2[X]$$. Moreover, no matter what irreducible degree $$8$$ polynomial we choose for $$f(X)$$, we'll get the same field up to isomorphism.

To make this a little more concrete, one element of $$GF(2^8)$$ might be given by the polynomial

$$g(X) = X^7 + X^4 + X^2.$$

and another might be given by the polynomial

$$h(X) = X^6 + X^2 + 1.$$

To keep things simple, we'll tend to conflate an element of our field with the polynomial of degree $$< 8$$ representing it.

We can add $$g$$ and $$h$$ (remember the coefficients are mod 2) in $$\operatorname{GF}(2^8)$$ to get 

$$\begin{align*}
    g(X) + h(X) &= X^7 + X^6 + X^4 + 2X^2 + 1\\
    &= X^7 + X^6 + X^4 + 1.
\end{align*}$$ 

To multiply them, we can perform the multiplication in $$\mathbb{Z}_2[X]$$ and then reduce mod $$f(X)$$:

$$\begin{align*}
    g(X) \cdot h(X) &= X^{13} + X^{10} + X^9 + X^8 + X^7 + X^6 + X^2\\
    &\rightsquigarrow X^7 + X^6 + X^3.
\end{align*}$$

The question is, how do we perform this reduction algorithmically? And how do we compute inverses in this strange field?

Operations in $$\small{\operatorname{GF}(2^8)}$$
---

Let's start with how to perform reduction.

All those proofs we brushed under the rug
---

Oh... we forgot all those proofs. Oops. Well, if you're curious, here are the promised explanations. Clearly they weren't important, but if you're curious they're kinda cute.

---

<small>**$$\operatorname{GF}(2^8)$$ is a field:** To see that $$\operatorname{GF}(2^8)$$ is a field, it sufficies to show that $$(f(X))$$ is a maximal ideal in $$\mathbb{Z}_2[X]$$. This is true, because if $$(f(X))$$ weren't maximal, it would be contained in some maximal ideal $$\mathfrak{m}$$ containing $$g(X) \notin (f(X)).$$ But since $$f(X)$$ is irreducible, it is coprime to any such polynomial $$g(X)$$. By Bezout's lemma, we can find $$a(X), b(X) \in \mathbb{Z}_2[X]$$ such that</small> 

$$\small{a(X) f(X) + b(X) g(X) = 1.}$$

<small>But then since $$f(X), g(X) \in \mathfrak{m}$$, we have $$1 \in \mathfrak{m}$$, a contradiction. Thus $$\operatorname{GF}(2^8)$$ is a field.</small>

<small>**Elements of $$\operatorname{GF}(2^8)$$ are represented by polynomials of degree $$< 8$$:** Next, we claim any element of this ring can be uniquely represented by a polynomial of degree $$< 8$$. Indeed, if $$g(X) \in \mathbb{Z}_2[X]$$ is a representative of some element of $$\operatorname{GF}(2^8)$$, then we can use polynomial long division to write 

$$\small{g(X) = a(X) f(X) + r(X),}$$

<small>where $$a(x), r(x) \in \mathbb{Z}_2[X]$$ and $$\operatorname{deg} r(x) < 8.$$ Since $$g(X)$$ and $$r(X)$$ differ by a multiple of $$f(X)$$, then $$r(X)$$ is also a representative of our element of $$\operatorname{GF}(2^8)$$. This shows our claim.</small>

<small>**These representatives are unique:** Moreover, we claim any two elements with distinct representatives of degree $$< 8$$ are distinct. If this wasn't the case, we'd have equal elements of $$\operatorname{GF}(2^8)$$ with representatives $$g(X), h(X)$$ which are distinct but both have degree $$< 8.$$ Subtracting $$g$$ and $$h$$, we would get a representative $$g(X) - h(X) \neq 0$$ of $$0 \in \operatorname{GF}(2^8)$$ with degree $$< 8$$. But this is impossible, as the quotient map</small> 

$$\small{\mathbb{Z}_2[X] \to \mathbb{Z}_2[X] / (f(X))}$$

<small>only sends multiples of $$f(X)$$ to $$0$$, meaning $$g(X) - h(X)$$ can't be a representative of $$0$$.</small>

<small>**$$\operatorname{GF}(2^8)$$ is well defined:** Lastly, we want to show that our choise of $$f(X)$$ doesn't matter. It is sufficient to show the stronger statement that all finite fields of order $$2^8$$ are isomorphic. Take any field $$F$$ of order $$2^8$$. $$F$$ must have characteristic $$2$$ (as $$F$$'s characteristic must divide its order). We claim that $$F$$ is isomorphic to the splitting field of</small>

$$\small{h(X) = X^{2^8} - X.}$$

<small>It sufficies to show that every element of $$F$$ is a root of $$h(X)$$, as then $$h(X)$$ must split into</small>

$$\small{h(X) = \prod_{a \in F} (x - a)}.$$

<small>And indeed, $$0 \in F$$ is clearly a root of $$h(X)$$, and any other $$a \in F^\times$$ must have order dividing $$\lvert F^\times \rvert = 2^8 - 1,$$ meaning $$a$$ is a root of $$X^{2^8 - 1} - 1$$ and thus also a root of $$h(X) = X^{2^8} - X.$$ Since splitting fields are unique up to isomorphism, we have all fields of order $$2^8$$ are isomorphic.</small>


---

