---
layout: post
title: An introduction to BCH codes
date: 2020-11-02 09:00
summary: An introduction to Bose–Chaudhuri–Hocquenghem (BCH) cyclic error correcting codes, aimed at engineers who want to understand the background without all of the mathematical rigor. This blog also assumes familiarity with regular linear error correcting codes (like Hamming codes), so I won't go over any general background on error correcting codes.
categories: blogs
---

I've been working with error correcting codes, and I had a hard time finding good, consolidated sources online which explained the background at a level of detail that I was comfortable with. Things were either very high level, which didn't give enough details to implement a real solution, or overly theoretical, which doesn't appeal to anyone not accustomed to the theorem, proof, lemma, proof style of graduate math textbooks. What I hope to do here is give an explanation of cyclic error correcting codes and BCH codes, aimed at engineers and applied scientists who want to understand an implementation and get a flavor of the theory, without going into proofs. For this reason, I'll just state results without proving them. Instead, I will try to give an outline of how to implement each part of the process. Those interested in more complete picture are encouraged refer to [Costello and Lin](https://books.google.ca/books/about/Error_Control_Coding.html?id=autQAAAAMAAJ&redir_esc=y).

This blog also assumes familiarity with regular linear error correcting codes (like Hamming codes), so I won't go over any general background on error correcting codes.

The blog will be divided into the following sections:
1. Background on finite fields
2. Introduction to binary cyclic linear error correcting codes
3. Introduction to BCH codes
4. Encoding BCH codes
5. Decoding BCH codes

## Finite fields

The mathematics of cyclic error correcting codes is based on finite fields, so it's impossible to have a meaningful discussion of cyclic codes without first giving the necessary background on finite fields. We'll try to cover the minimum necessary material to understand future sections, with examples along the way to aid understanding. Let's get started!

**Finite fields**, or **Galois fields**, written <img src="https://latex.codecogs.com/gif.latex?GF(q)"/>, are fields (i.e. they are closed under, and have inverses for, addition and multiplication) which have a finite number of elements. The simplest examples are the integers modulo a prime -- <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> is the field <img src="https://latex.codecogs.com/gif.latex?\{0,1\}"/> with addition and multiplication modulo 2, <img src="https://latex.codecogs.com/gif.latex?GF(3)"/> is the field <img src="https://latex.codecogs.com/gif.latex?\{0, 1, 2\}"/> with addition and multiplication modulo 3, etc. To aid understanding, let's write out the addition and multiplication table for <img src="https://latex.codecogs.com/gif.latex?GF(5)"/>. When adding or multiplying, everything is modulo 5, so 4 + 4 = 8 mod 5 = 3, and 4 * 3 = 12 mod 5 = 2 for example.

The addition table is:

| +  | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| **0** | 0 | 1 | 2 | 3 | 4 |
| **1** | 1 | 2 | 3 | 4 | 0 |
| **2** | 2 | 3 | 4 | 0 | 1 |
| **3** | 3 | 4 | 0 | 1 | 2 |
| **4** | 4 | 0 | 1 | 2 | 3 |

The multiplication table is:

| x  | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| **0** | 0 | 0 | 0 | 0 | 0 |
| **1** | 0 | 1 | 2 | 3 | 4 |
| **2** | 0 | 2 | 4 | 1 | 3 |
| **3** | 0 | 3 | 1 | 4 | 2 |
| **4** | 0 | 4 | 3 | 2 | 1 |

Galois fields must have a number of elements (order) equal to a prime power, and all Galois fields with the same order are isomorphic to each other. To construct a Galois field with non-prime number of elements (i.e. a prime power number of elements), or an **extension field**, we use polynomials instead of integers to represent field elements. To construct <img src="https://latex.codecogs.com/gif.latex?GF(16) = GF(2^4)"/>, we use <img src="https://latex.codecogs.com/gif.latex?GF(16) = GF(2)[x]"/> modulo P, written <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/P"/>. <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]"/> is pronounced "GF(2) adjoin x", and represents the set of all polynomials whose coefficients are in <img src="https://latex.codecogs.com/gif.latex?GF(2)"/>, and <img src="https://latex.codecogs.com/gif.latex?P"/>
is an **irreducible polynomial** of degree 2 over <img src="https://latex.codecogs.com/gif.latex?GF(2)"/>. Irreducible polynomials are polynomials which cannot be factored. For example, in <img src="https://latex.codecogs.com/gif.latex?GF(2)"/>, <img src="https://latex.codecogs.com/gif.latex?x^2+x+1"/> is irreducible, but <img src="https://latex.codecogs.com/gif.latex?x^2+1"/> is not, since <img src="https://latex.codecogs.com/gif.latex?(x+1)(x+1)=x^2+2x+1"/>, but 2=0, so <img src="https://latex.codecogs.com/gif.latex?(x+1)(x+1)=x^2+1"/>. The irreducible polynomials of the first few orders for <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> are:

| Order | Irreducible Polynomial                                                                     |
|-------|--------------------------------------------------------------------------------------------|
| 2     | <img src="https://latex.codecogs.com/gif.latex?x^2+x+1"/>                                                                                   |
| 3     | <img src="https://latex.codecogs.com/gif.latex?x^3+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^3+x^2+1  "/>                                                                       |
| 4     | <img src="https://latex.codecogs.com/gif.latex?x^4+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^4+x^3+x^2+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^4+x^3+1 "/>                                                       |
| 5     | <img src="https://latex.codecogs.com/gif.latex?x^5+x^2+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^3+x^2+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^3+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^4+x^3+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^4+x^3+x^2+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^4+x^2+x+1"/> |

There are, in general, multiple irreducible polynomials of each degree for each base field. In practice, we choose the
one that has the smallest weight (so we prefer to use <img src="https://latex.codecogs.com/gif.latex?x^3+x+1"/> instead of <img src="https://latex.codecogs.com/gif.latex?x^3+x^2+1  "/>, for example). Since all finite fields of an order are isomorphic, constructing an extension field of a particular order with any valid irreducible polynomial creates an isomorphic finite field (the addition and multiplication tables will just have different entries). Let's do an example of constructing an extension field <img src="https://latex.codecogs.com/gif.latex?GF(9)"/> from the base field <img src="https://latex.codecogs.com/gif.latex?GF(3)"/>. We choose <img src="https://latex.codecogs.com/gif.latex?P = x^2 + 1"/>, which is an irreducible polynomial of order 2 in <img src="https://latex.codecogs.com/gif.latex?GF(3)[x]"/>. <img src="https://latex.codecogs.com/gif.latex?GF(3)[x]/(x^2+1)"/> is then the set of all polynomials with coefficients in <img src="https://latex.codecogs.com/gif.latex?GF(3)"/> modulo <img src="https://latex.codecogs.com/gif.latex?x^2+1"/>, or the set <img src="https://latex.codecogs.com/gif.latex?\{0, 1, 2, x, x+1, x+2, 2x, 2x+1, 2x+2\}"/>, which has 9 elements. Addition and multiplication using this polynomial representation is modulo <img src="https://latex.codecogs.com/gif.latex?x^2+1"/> and modulo 3. Again, for understanding, let's write out the multiplication table (the addition table is easier to calculate) for <img src="https://latex.codecogs.com/gif.latex?GF(3)[x]/(x^2+1)"/> and work out some specific examples.

The multiplication table is:

|   x   | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?x+2"/>  | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> |
|------|---|------|------|------|------|------|------|------|------|
| <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/>    |
| <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?x+2"/>  | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> |
| <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> |
| <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?x+2"/>  | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> |
| <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?x+2"/>  | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?x"/>    |
| <img src="https://latex.codecogs.com/gif.latex?x+2"/>  | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?x+2"/> | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?2"/>    |
| <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?x+2"/>  |
| <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?x+2"/>  | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?2x"/>   | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?1"/>    |
| <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?0"/> | <img src="https://latex.codecogs.com/gif.latex?2x+2"/> | <img src="https://latex.codecogs.com/gif.latex?x+1"/>  | <img src="https://latex.codecogs.com/gif.latex?2x+1"/> | <img src="https://latex.codecogs.com/gif.latex?x"/>    | <img src="https://latex.codecogs.com/gif.latex?2"/>    | <img src="https://latex.codecogs.com/gif.latex?x+2"/>  | <img src="https://latex.codecogs.com/gif.latex?1"/>    | <img src="https://latex.codecogs.com/gif.latex?2x"/>   |

Here are some randomly selected worked out examples so you can get a feel for modular arithmetic on polynomials. For each of these, remember that <img src="https://latex.codecogs.com/gif.latex?x^2+1=0"/> and <img src="https://latex.codecogs.com/gif.latex?3=0"/>, so adding <img src="https://latex.codecogs.com/gif.latex?3x"/> or <img src="https://latex.codecogs.com/gif.latex?3"/> is the same as adding 0. I also want to reduce <img src="https://latex.codecogs.com/gif.latex?x^2+1"/> to 0 to get rid of <img src="https://latex.codecogs.com/gif.latex?x^2"/> terms, since the result of an operation should always be a member of the field (closure) and the elements in the field only go up to <img src="https://latex.codecogs.com/gif.latex?x^1"/>.

<img src="https://latex.codecogs.com/gif.latex?x(x+2) = x^2+2x = x^2+3+2x = (x^2+1)+2x+2 = 2x+2"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?2x(x+1) = 2x^2+2x = 2x^2+3+2x = 2(x^2+1)+2x+1 = 2x+1"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)(x+2) = x^2+3x+2 = x^2+2 = (x^2+1)+1 = 1"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?(2x)(2x) = 4x^2 = 4x^2 + 6 = 4(x^2+1)+2 = 2"/> <br/>

To someone who has done regular boring arithmetic their whole life, modular arithmetic can seem strange and unwieldy at first, but you may find it more enjoyable once you get the hang of it! 

A **primitive element**, <img src="https://latex.codecogs.com/gif.latex?\alpha"/> , of a Galois field, is an element whose successive powers generate all elements of the field except 0. In other words, it is a primitive (n-1)th root of unity, where n is the number of the elements in the field. Finding primitive elements is a matter of brute force: try all elements other than 0 and 1. This is easier than it sounds though, since about a third of the elements in a finite field are primitive elements.
For example, <img src="https://latex.codecogs.com/gif.latex?\alpha=2"/> is a primitive element of <img src="https://latex.codecogs.com/gif.latex?GF(3)"/> and <img src="https://latex.codecogs.com/gif.latex?GF(5)"/>, but not <img src="https://latex.codecogs.com/gif.latex?GF(7)"/>. <img src="https://latex.codecogs.com/gif.latex?\alpha=x+1"/> is a primitive element of <img src="https://latex.codecogs.com/gif.latex?GF(3)[x]/(x^2+1)"/>. To see why, let's do the multiplication, which is made easy by jumping to the correct cell in the multiplication table above.

<img src="https://latex.codecogs.com/gif.latex?(x+1)^1 = x+1"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)^2 = 2x"/><br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)^3 = 2x+1"/><br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)^4 = 2"/><br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)^5 = 2x+2"/><br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)^6 = x"/><br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)^7 = x+2"/><br/>
<img src="https://latex.codecogs.com/gif.latex?(x+1)^8 = 1"/><br/>

We've been using the base field <img src="https://latex.codecogs.com/gif.latex?GF(3)"/> in this section to give an example with field order higher than 2. In the remainder of this blog though, we'll use <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> as the base field in examples, since ultimately we're interested in binary codes, for which <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> is always the base field.

There is a unique polynomial <img src="https://latex.codecogs.com/gif.latex?m(\alpha)"/> over <img src="https://latex.codecogs.com/gif.latex?GF(q)[x]"/> for each element <img src="https://latex.codecogs.com/gif.latex?\alpha"/> in <img src="https://latex.codecogs.com/gif.latex?GF(q^m)"/> which is the smallest polynomial possible, and such that <img src="https://latex.codecogs.com/gif.latex?m(\alpha) = 0"/>, called a **minimal polynomial**. A **primitive polynomial** is the minimal polynomial of a primitive element. As an example, let's construct the finite field <img src="https://latex.codecogs.com/gif.latex?GF(32)"/>, or <img src="https://latex.codecogs.com/gif.latex?GF(2^5)"/>. We use <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/(x^5+x^2+1)"/>. Some examples of
elements are: 0, 1, <img src="https://latex.codecogs.com/gif.latex?x"/>, <img src="https://latex.codecogs.com/gif.latex?x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^2"/>, <img src="https://latex.codecogs.com/gif.latex?x^2+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^2+x"/>, etc. For a primitive element <img src="https://latex.codecogs.com/gif.latex?\alpha"/>, we can generate each element of
<img src="https://latex.codecogs.com/gif.latex?GF(32)"/> by successive multiplication by <img src="https://latex.codecogs.com/gif.latex?\alpha"/>, as shown below. We use <img src="https://latex.codecogs.com/gif.latex?z"/> instead of <img src="https://latex.codecogs.com/gif.latex?x"/> as the symbol for finite field polynomial terms because we will start reserving <img src="https://latex.codecogs.com/gif.latex?x"/> for minimal polynomials.

<img src="https://latex.codecogs.com/gif.latex?\alpha^1 = z"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^2 = z^2"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^3 = z^3"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^4 = z^4"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^5 = z^2 + 1"/> (since we're in modulo 2 and modulo <img src="https://latex.codecogs.com/gif.latex?x^5+x^2+1"/>)<br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^6 = z^3+z"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^7 = z^4+z^2"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^8 = z^5+z^3 = z^3+z^2+1"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^9 = z^4+z^3+1"/><br/>
...and so forth, until<br/>
<img src="https://latex.codecogs.com/gif.latex?\alpha^{31} = 1"/><br/>

Each <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> has a minimal polynomial associated with it, and each minimal polynomial has at least one
<img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> as a root. They can be found through the fact that if <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> is a root of a minimal polynomial over the base field <img src="https://latex.codecogs.com/gif.latex?GF(p)"/>, then so is
<img src="https://latex.codecogs.com/gif.latex?\alpha^{i*p}"/>. So for our example using <img src="https://latex.codecogs.com/gif.latex?GF(32)"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^2"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^4"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^8"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^{16}"/> all share the same minimal polynomial, and so do <img src="https://latex.codecogs.com/gif.latex?\alpha^3"/>,
<img src="https://latex.codecogs.com/gif.latex?\alpha^6"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^{12}"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^{24}"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^{17}"/> (17 = 48 mod 31). To find the minimal polynomial for <img src="https://latex.codecogs.com/gif.latex?\alpha"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^2"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^4"/>,
<img src="https://latex.codecogs.com/gif.latex?\alpha^8"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^{16}"/>, we can calculate <img src="https://latex.codecogs.com/gif.latex?(x-z)(x-z^2)(x-z^4)(x-z^3-z^2-1)(...)"/> and simplify. All the <img src="https://latex.codecogs.com/gif.latex?z"/>'s should cancel out and we should be left with a polynomial in <img src="https://latex.codecogs.com/gif.latex?x"/> only. We'll go over a more explicit example where minimal polynomials are computed in Section 3.

## Linear binary cyclic codes

Next, let's discuss linear binary cyclic codes.

- **Linear**: The weighted sum of any two codewords is also a codeword

- **Binary**: The codeword symbols are 0 and 1

- **Cyclic**: Any cyclic shift of a codeword is another codeword

For example, all sets of valid 4-length linear binary cyclic codes are:

- {0000, 1111}
- {0000, 0101, 1010, 1111}
- {0000, 0110, 0011, 1100, 0001, 0010, 0100, 1000, 0101, 1010, 1001, 1111, 0111, 1110, 1011, 1101}

Binary linear cyclic codes are represented by <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/(x^n-1)"/>. In this representation, multiplying by <img src="https://latex.codecogs.com/gif.latex?x"/> amounts to
a cyclic left-shift by 1, since <img src="https://latex.codecogs.com/gif.latex?x^n=1"/>. Extension fields of <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> naturally work to represent sequences of binary numbers, since their coefficients are either 0 or 1, so the resulting polynomials can be thought of as binary messages with the most significant bit (MSB) at either the highest degree of <img src="https://latex.codecogs.com/gif.latex?x"/> or lowest degree of <img src="https://latex.codecogs.com/gif.latex?x"/>.

Furthermore, this means that ANY polynomial in <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/(x^n-1)"/> multiplied by a valid codeword is another valid
codeword, since valid codewords are some subset of <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/(x^n-1)"/>, and multiplication by an arbitrary polynomial is a linear combination of cyclic shifts of the original codeword. Since all shifted codewords are valid codewords and all linear combinations
of codewords are valid codewords, the resulting polynomial must be a valid codeword.

The question then, is **how do we generate codewords**? I.e. how do we choose a generator polynomial which maps all valid
messages to all valid codewords? Furthermore, how do we DESIGN the generator polynomial so that we get the block length, message length and error correction capability that we want?

For a cyclic code C(n,k)
- The generator polynomial <img src="https://latex.codecogs.com/gif.latex?g(x)"/> is monic (its highest power has coefficient 1)
- C consists of all multiples of <img src="https://latex.codecogs.com/gif.latex?g(x)"/> with polynomials of degree k-1 or less
- <img src="https://latex.codecogs.com/gif.latex?g(x)"/> is a factor of <img src="https://latex.codecogs.com/gif.latex?x^n-1"/>

The third point is the most important, and means that we are in the business of factoring <img src="https://latex.codecogs.com/gif.latex?x^n-1"/>. We want the prime
factors of <img src="https://latex.codecogs.com/gif.latex?x^n-1"/> in <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]"/>, and <img src="https://latex.codecogs.com/gif.latex?g(x)"/> is the polynomial with the smallest degree which generates the codeword
polynomials we want. So if we have codewords of degree n-1 and messages of degree k-1, then <img src="https://latex.codecogs.com/gif.latex?g(x)"/> must have degree n-k.

For example, all possible cyclic codeword polynomials of length three reside in <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/(x^3-1)"/>. <img src="https://latex.codecogs.com/gif.latex?x^3-1=x^3+1"/> factors into <img src="https://latex.codecogs.com/gif.latex?(x+1)(x^2+x+1)"/>, so the possible generator polynomials are:

<img src="https://latex.codecogs.com/gif.latex?g(x) = 1"/> (degree 0) <br/>
<img src="https://latex.codecogs.com/gif.latex?g(x) = x+1"/> (degree 1) <br/>
<img src="https://latex.codecogs.com/gif.latex?g(x) = x^2+x+1"/> (degree 2) <br/>
<img src="https://latex.codecogs.com/gif.latex?g(x) = (x+1)(x^2+x+1) = x^3+1"/> (degree 3) <br/>

- for <img src="https://latex.codecogs.com/gif.latex?g(x) = 1"/>, the degree of the message is 2. This is the trivial identity mapping, which maps all codewords of length 3 to themselves.
- for <img src="https://latex.codecogs.com/gif.latex?g(x) = x+1"/>, the degree of the message is 1. This is a mapping between messages of length 2 and codewords of length 3
- for <img src="https://latex.codecogs.com/gif.latex?g(x) = x^2+x+1"/>, the degree of the message is 0. This is a mapping between messages of length 1 and codewords of length 3. There are only two messages of length 1: 0, and 1, and only two codewords: 000, 111, so this is a repetition code!
- for <img src="https://latex.codecogs.com/gif.latex?g(x) = x^3+1"/>, because of the construction of the extension field, <img src="https://latex.codecogs.com/gif.latex?x^3+1 = 0"/>, so everything gets trivially mapped to 0.

The beauty and usefulness of creating codes in this way is that decoding a received code has the following properties:

For a received codeword <img src="https://latex.codecogs.com/gif.latex?v(x)"/>,

<img src="https://latex.codecogs.com/gif.latex?v(x) = i(x)g(x) + e(x)"/>

where <img src="https://latex.codecogs.com/gif.latex?i(x)"/> is the true message, and <img src="https://latex.codecogs.com/gif.latex?e(x)"/> is the error. Since we know which <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> we constructed <img src="https://latex.codecogs.com/gif.latex?g(x)"/> with, <img src="https://latex.codecogs.com/gif.latex?g(x)"/> is 0 at any of those <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/>'s. So

<img src="https://latex.codecogs.com/gif.latex?v(\alpha^i) = e(\alpha^i)"/>

furthermore,

<img src="https://latex.codecogs.com/gif.latex?v(x)"/> mod <img src="https://latex.codecogs.com/gif.latex?g(x)"/> = <img src="https://latex.codecogs.com/gif.latex?e(x)"/> mod <img src="https://latex.codecogs.com/gif.latex?g(x)"/>, since <img src="https://latex.codecogs.com/gif.latex?i(x)g(x)"/> mod <img src="https://latex.codecogs.com/gif.latex?g(x)"/> = 0. These two facts help us locate and correct errors.

## BCH Codes

BCH codes give an easy and useful way of defining <img src="https://latex.codecogs.com/gif.latex?g(x)"/> given a blocklength and a number of error correction bits.
Recall that, in order to find valid generator polynomials, we are looking for factors of <img src="https://latex.codecogs.com/gif.latex?x^n-1"/>. To link the factorization of <img src="https://latex.codecogs.com/gif.latex?x^n-1"/> with finite fields, we use the fact that the roots of <img src="https://latex.codecogs.com/gif.latex?x^{q-1} - 1"/> are the non-zero elements of <img src="https://latex.codecogs.com/gif.latex?GF(q)"/>!

In other words, the roots of <img src="https://latex.codecogs.com/gif.latex?x^{q-1}-1"/> form a finite field of order q if 0 is added. So for elements <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> of <img src="https://latex.codecogs.com/gif.latex?GF(q)"/>,
<img src="https://latex.codecogs.com/gif.latex?x^{q-1}-1 = (x-\alpha^1)(x-\alpha^2)...(x-\alpha^{q-1})"/>

We have already seen that some of the factors of <img src="https://latex.codecogs.com/gif.latex?x^n-1"/> combine to form prime factors, and this is true here as well.
Each element of <img src="https://latex.codecogs.com/gif.latex?GF(q)"/> is the root of EXACTLY ONE of the prime factors of <img src="https://latex.codecogs.com/gif.latex?x^{q-1}-1"/>. Therefore, <img src="https://latex.codecogs.com/gif.latex?g(x)"/> must be the least common multiple (LCM) of some of the minimal polynomials of <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/>. Which ones depends on the code we are trying to create. The key concept of primitive, narrow-sense BCH codes is that, **for t bits of desired error correcting capability, the generator polynomial is the lowest degree polynomial with** <img src="https://latex.codecogs.com/gif.latex?\alpha^1, \alpha^2, ..., \alpha^{2t}"/> **as its roots.**

The steps to construct a BCH code are:
1. For a given blocklength <img src="https://latex.codecogs.com/gif.latex?n=q-1"/> with <img src="https://latex.codecogs.com/gif.latex?q=2^m"/>, construct a finite field <img src="https://latex.codecogs.com/gif.latex?GF(q)"/>
2. Find the minimal polynomials of the elements of <img src="https://latex.codecogs.com/gif.latex?GF(q)"/>
3. The generator polynomial <img src="https://latex.codecogs.com/gif.latex?g(x)"/> is <img src="https://latex.codecogs.com/gif.latex?LCM\{m_1(x), m_2(x), ..., m_2t(x)\}"/>, where t is the number of error correcting bits
desired. The Hamming distance is d = 2t+1
4. The message length is k = n - (degree(<img src="https://latex.codecogs.com/gif.latex?g(x)"/>) + 1). This creates an (n, k, d) BCH code.

Let's do an example for a code of blocklength <img src="https://latex.codecogs.com/gif.latex?15 = 2^4-1"/>. This requires us to construct the extension field <img src="https://latex.codecogs.com/gif.latex?GF(16)"/>, which we can do with the irreducible polynomial <img src="https://latex.codecogs.com/gif.latex?x^4+x+1"/>. So our extension field is <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/(x^4+x+1)"/>. The field elements are:

| element  | polynomial  | binary |
|----------|-------------|--------|
| <img src="https://latex.codecogs.com/gif.latex?0"/>        | <img src="https://latex.codecogs.com/gif.latex?0"/>           | 0000   |
| <img src="https://latex.codecogs.com/gif.latex?1"/>        | <img src="https://latex.codecogs.com/gif.latex?1"/>           | 0001   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha"/>    | <img src="https://latex.codecogs.com/gif.latex?z"/>           | 0010   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^2"/>  | <img src="https://latex.codecogs.com/gif.latex?z^2"/>         | 0100   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^3"/>  | <img src="https://latex.codecogs.com/gif.latex?z^3"/>         | 1000   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^4"/>  | <img src="https://latex.codecogs.com/gif.latex?z+1"/>         | 0011   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^5"/>  | <img src="https://latex.codecogs.com/gif.latex?z^2+z"/>       | 0110   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^6"/>  | <img src="https://latex.codecogs.com/gif.latex?z^3+z^2"/>     | 1100   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^7"/>  | <img src="https://latex.codecogs.com/gif.latex?z^3+z+1"/>     | 1011   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^8 "/> | <img src="https://latex.codecogs.com/gif.latex?z^2+1"/>       | 0101   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^9 "/> | <img src="https://latex.codecogs.com/gif.latex?z^3+z"/>       | 1010   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{10}"/> | <img src="https://latex.codecogs.com/gif.latex?z^2+z+1"/>     | 0111   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{11}"/> | <img src="https://latex.codecogs.com/gif.latex?z^3+z^2+z"/>   | 1110   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{12}"/> | <img src="https://latex.codecogs.com/gif.latex?z^3+z^2+z+1"/> | 1111   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{13}"/> | <img src="https://latex.codecogs.com/gif.latex?z^3+z^2+1"/>   | 1101   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{14}"/> | <img src="https://latex.codecogs.com/gif.latex?z^3+1"/>       | 1001   |

(note, <img src="https://latex.codecogs.com/gif.latex?\alpha^{15}=1"/>)

The first minimal polynomial <img src="https://latex.codecogs.com/gif.latex?m_1(x)"/> is associated with <img src="https://latex.codecogs.com/gif.latex?\alpha, \alpha^2, \alpha^4, \alpha^8"/>, so

<img src="https://latex.codecogs.com/gif.latex?m_1(x) = (x-z)(x-z^2)(x-z-1)(x-z^2-1) = x^4+x+1"/>

similarly, <img src="https://latex.codecogs.com/gif.latex?m_3(x)"/> is associated with <img src="https://latex.codecogs.com/gif.latex?\alpha^3, \alpha^6, \alpha^{12}, \alpha^9"/> (24-15=9), so

<img src="https://latex.codecogs.com/gif.latex?m_3(x) = (x-z^3)(x-z^3-z^2)(x-z^3-z)(x-z^3-z^2-z-1) = x^4+x^3+x^2+x+1"/>

and so forth:

<img src="https://latex.codecogs.com/gif.latex?m_5(x) = (x-z^2-z)(x-z^2-z-1) = x^2+x+1"/>

<img src="https://latex.codecogs.com/gif.latex?m_7(x) = (x-z^3-z-1)(x-z^3-1)(x-z^3-z^2-1)(x-z^3-z^2-z) = x^4+x^3+1"/>

So the minimal polynomials are

| element  | minimal polynomial    |
|----------|-----------------------|
| <img src="https://latex.codecogs.com/gif.latex?\alpha"/>    | <img src="https://latex.codecogs.com/gif.latex?x^4+x+1 = m_1"/>         |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^2"/>  | <img src="https://latex.codecogs.com/gif.latex?m_1"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^3"/>  | <img src="https://latex.codecogs.com/gif.latex?x^4+x^3+x^2+x+1 = m_3"/> |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^4"/>  | <img src="https://latex.codecogs.com/gif.latex?m_1"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^5"/>  | <img src="https://latex.codecogs.com/gif.latex?x^2+x+1 = m_5"/>         |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^6"/>  | <img src="https://latex.codecogs.com/gif.latex?m_3"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^7"/>  | <img src="https://latex.codecogs.com/gif.latex?x^4+x^3+1 = m_7"/>       |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^8"/>  | <img src="https://latex.codecogs.com/gif.latex?m_1"/>                   |
|<img src="https://latex.codecogs.com/gif.latex?\alpha^9"/>  | <img src="https://latex.codecogs.com/gif.latex?m_3"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{10}"/> | <img src="https://latex.codecogs.com/gif.latex?m_5"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{11}"/> | <img src="https://latex.codecogs.com/gif.latex?m_7"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{12}"/> | <img src="https://latex.codecogs.com/gif.latex?m_3"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{13}"/> | <img src="https://latex.codecogs.com/gif.latex?m_7"/>                   |
| <img src="https://latex.codecogs.com/gif.latex?\alpha^{14}"/> | <img src="https://latex.codecogs.com/gif.latex?m_7"/>                   |

A valid <img src="https://latex.codecogs.com/gif.latex?g(x)"/> is found by choosing t and computing <img src="https://latex.codecogs.com/gif.latex?LCM\{m_1(x), ..., m_{2t}(x)\}"/>. For example:
- For t=1, we get <img src="https://latex.codecogs.com/gif.latex?g(x) = LCM\{m_1(x), m_2(x)\} = m_1(x)"/>, and <img src="https://latex.codecogs.com/gif.latex?g(x)"/> has degree 4. For block length 15, this gives a (15, 11, 3) BCH code with one bit of error correcting capability. 
- For t=2, we get <img src="https://latex.codecogs.com/gif.latex?g(x) = LCM\{m_1(x), m_2(x), m_3(x), m_4(x)\} = m_1(x)m_3(x)"/>, and <img src="https://latex.codecogs.com/gif.latex?g(x)"/> has degree 8. This gives a (15, 7, 5) BCH code with two bits of error correcting capability. 
- For t=3, we get <img src="https://latex.codecogs.com/gif.latex?g(x) = m_1(x)m_3(x)m_5(x)"/>, and <img src="https://latex.codecogs.com/gif.latex?g(x)"/> has degree 10. This gives a (15, 5, 7) BCH code, and corrects 3 errors. 
- For t=4, we get <img src="https://latex.codecogs.com/gif.latex?g(x) = m_1(x)m_3(x)m_5(x)m_7(x)"/>, and <img src="https://latex.codecogs.com/gif.latex?g(x)"/> has degree 14. This gives a (15, 1) repetition code with Hamming distance 15 and corrects 7 errors. 
- If we add the final <img src="https://latex.codecogs.com/gif.latex?(x+1)"/> factor, then we get <img src="https://latex.codecogs.com/gif.latex?x^{15}+1"/> as the generator polynomial. Since <img src="https://latex.codecogs.com/gif.latex?x^{15}+1=0"/>, this generator polynomial maps everything to 0, which is a trivial mapping. 
- Also, we could have used just 1 as the generator polynomial, which maps everything to itself (i.e. a (15, 15, 0) code), which is another trivial mapping.

Codes constructed this way are called primitive (i.e. only block lengths of <img src="https://latex.codecogs.com/gif.latex?2^m-1"/> are used) narrow-sense (i.e. we start with <img src="https://latex.codecogs.com/gif.latex?\alpha^1"/> instead of another <img src="https://latex.codecogs.com/gif.latex?\alpha"/>) BCH codes.

## Encoding BCH codes

BCH codes can be constructed **non-systematically**, by simply multiplying a message polynomial by the generator polynomial,
or **systematically**, by embedding the message itself in the codeword. The only requirement is that the transmitted
codeword is divisible by <img src="https://latex.codecogs.com/gif.latex?g(x)"/>. In the first case, we have

<img src="https://latex.codecogs.com/gif.latex?v(x) = g(x)i(x)"/>

In the second case, we have

<img src="https://latex.codecogs.com/gif.latex?v(x) = i(x)x^{n-k} + r(x)"/>, where <img src="https://latex.codecogs.com/gif.latex?r(x) = i(x)x^{n-k}"/> mod <img src="https://latex.codecogs.com/gif.latex?g(x)"/>

so, <img src="https://latex.codecogs.com/gif.latex?i(x)"/> gets shifted to the MSB of <img src="https://latex.codecogs.com/gif.latex?v(x)"/>, and then the leftover bits get modified to make <img src="https://latex.codecogs.com/gif.latex?v(x)"/> divisible by <img src="https://latex.codecogs.com/gif.latex?g(x)"/>.

Non-systematic encoding is trivial to implement: just multiply the message by the generator and simplify. To implement systematic encoding, we shift the message polynomial so that its MSB is at the MSB of the codeword, and then modify the zeros at the LSB of the codeword (of length n-k) in a way so that the resulting codeword
is divisible by <img src="https://latex.codecogs.com/gif.latex?g(x)"/>. Practically this looks like the following:

For a codeword represented by a polynomial <img src="https://latex.codecogs.com/gif.latex?v(x)"/> of degree n, and message represented by a polynomial <img src="https://latex.codecogs.com/gif.latex?i(x)"/> of degree k, the generator polynomial <img src="https://latex.codecogs.com/gif.latex?g(x)"/> is of degree n-k, so:

1. We multiply the message polynomial <img src="https://latex.codecogs.com/gif.latex?i(x)"/> by <img src="https://latex.codecogs.com/gif.latex?x^{n-k}"/> to shift it to MSB position in the codeword polynomial <img src="https://latex.codecogs.com/gif.latex?v(x)"/>
2. <img src="https://latex.codecogs.com/gif.latex?v(x)"/> must be divisible by <img src="https://latex.codecogs.com/gif.latex?g(x)"/> to be a valid codeword, so it can be written as <img src="https://latex.codecogs.com/gif.latex?q(x)g(x)"/> for some <img src="https://latex.codecogs.com/gif.latex?q(x)"/> to be determined
3. The difference between <img src="https://latex.codecogs.com/gif.latex?q(x)g(x)"/> and <img src="https://latex.codecogs.com/gif.latex?i(x)x^{n-k}"/> is the remainder <img src="https://latex.codecogs.com/gif.latex?r(x)"/>, which we can compute by taking
<img src="https://latex.codecogs.com/gif.latex?i(x)x^{n-k}"/> and finding the modulo with <img src="https://latex.codecogs.com/gif.latex?g(x)"/>
4. The encoded codeword is then <img src="https://latex.codecogs.com/gif.latex?i(x)x^{n-k} + r(x)"/> (+ and - are the same in modulo 2 arithmetic). Since <img src="https://latex.codecogs.com/gif.latex?r(x)"/> is guaranteed to have degree less than n-k, it doesn't interfere with our k-degree message polynomial

So we need to find <img src="https://latex.codecogs.com/gif.latex?i(x)x^{n-k}"/> mod <img src="https://latex.codecogs.com/gif.latex?g(x)"/>, and set those bits in the codeword after shifting the message to th MSB.

Let's do an example of encoding a message using non-systematic and systematic encoding, using the (15, 11, 3) code that we computed previously. This code has <img src="https://latex.codecogs.com/gif.latex?g(x) = x^4+x+1"/>. Our message can be any 11-length binary number, or 10th degree polynomial. We'll choose 10100010001 as our message, or <img src="https://latex.codecogs.com/gif.latex?i(x)=x^{10} + x^8 + x^4 + 1"/>.

For non-systematic encoding, we simply multiply <img src="https://latex.codecogs.com/gif.latex?i(x)"/> with <img src="https://latex.codecogs.com/gif.latex?g(x)"/>, to get <img src="https://latex.codecogs.com/gif.latex?x^{14}+x^{12}+x^{11}+x^{10}+x^{9}+x^5+x+1"/>, or 101111000100011 in binary.

For systematic encoding, we start by multiplying <img src="https://latex.codecogs.com/gif.latex?i(x)"/> by <img src="https://latex.codecogs.com/gif.latex?x^4"/> to obtain <img src="https://latex.codecogs.com/gif.latex?x^{14} + x^{12}+x^8+x^4"/>. We then use polynomial division mod 2 to find the remainder after dividing by <img src="https://latex.codecogs.com/gif.latex?g(x)"/>, which happens to be zero (I never said I would choose a hard example -- you can check that <img src="https://latex.codecogs.com/gif.latex?g(x)(x^{10}+x^8+x^7+x^6+x^5+x^4)=x^{14} + x^{12}+x^8+x^4"/>). So the systematically encoded codeword is 101000100010000, which indeed contains the original message verbatim in the MSB of the codeword.

## Decoding BCH codes

Decoding BCH codes is more involved than encoding them, but ultimately follows a procedure which is tedious but sane, and easy to implement in both software and hardware. We start with the received polynomial <img src="https://latex.codecogs.com/gif.latex?r(x)"/>:

<img src="https://latex.codecogs.com/gif.latex?r(x)&space;=&space;r_0&space;&plus;&space;r_1&space;x&space;&plus;&space;r_2&space;x^2&space;&plus;&space;...&space;&plus;&space;r_{n-1}&space;x^{n-1}&space;=&space;v(x)&space;&plus;&space;e(x)" title="r(x) = r_0 + r_1 x + r_2 x^2 + ... + r_{n-1} x^{n-1} = v(x) + e(x)" />

where <img src="https://latex.codecogs.com/gif.latex?v(x)"/> is the transmitted codeword, and <img src="https://latex.codecogs.com/gif.latex?e(x)"/> is the error. The error for binary codes can be written

<img src="https://latex.codecogs.com/gif.latex?e(x) = x^{j_1} + x^{j_2} + ... + x^{j_\nu}"/>

where <img src="https://latex.codecogs.com/gif.latex?\{j_1, ... j_{\nu}\"/>} is the set of error locations. The sydromes <img src="https://latex.codecogs.com/gif.latex?S_i"/> are just <img src="https://latex.codecogs.com/gif.latex?r(x)"/> evaluated at <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> for
<img src="https://latex.codecogs.com/gif.latex?i\in \{1,2,...,2t\}"/>. An easier pen-and-paper way to evaluate the syndromes is to use the fact that, for a particular
<img src="https://latex.codecogs.com/gif.latex?\alpha^i"/>, if <img src="https://latex.codecogs.com/gif.latex?m_i(x)"/>, the minimal polynomial for <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> is known, then

<img src="https://latex.codecogs.com/gif.latex?r(x) = m_i(x)a_i(x) + b_i(x)"/>

for some <img src="https://latex.codecogs.com/gif.latex?a_i(x)"/>, and where <img src="https://latex.codecogs.com/gif.latex?b_i(x)"/> is <img src="https://latex.codecogs.com/gif.latex?r(x)"/> mod <img src="https://latex.codecogs.com/gif.latex?m_i(x)"/>. Since <img src="https://latex.codecogs.com/gif.latex?m_i(\alpha^i) = 0"/>, we can write

<img src="https://latex.codecogs.com/gif.latex?S_i = r(\alpha_i) = b_i(\alpha_i)"/>

For example, for a (15,7,5) BCH code, if r(x) = 000000100000001 or <img src="https://latex.codecogs.com/gif.latex?r(x) = x^8 + 1"/>, then since t=2, we need to
compute four syndromes, <img src="https://latex.codecogs.com/gif.latex?S_1"/>, <img src="https://latex.codecogs.com/gif.latex?S_2"/>, <img src="https://latex.codecogs.com/gif.latex?S_3"/>, and <img src="https://latex.codecogs.com/gif.latex?S_4"/>. We can do this by finding <img src="https://latex.codecogs.com/gif.latex?r(x)"/> mod <img src="https://latex.codecogs.com/gif.latex?m_i(x)"/> for <img src="https://latex.codecogs.com/gif.latex?i=1, 2, 3, 4"/>.

- For the first syndrome, <img src="https://latex.codecogs.com/gif.latex?m_1 = x^4+x+1"/>. <img src="https://latex.codecogs.com/gif.latex?b_1 = (x^8+1)"/> mod <img src="https://latex.codecogs.com/gif.latex?(x^4+x+1) = x^2"/>. <img src="https://latex.codecogs.com/gif.latex?S_1 = b_1(\alpha) = \alpha^2"/>
	- <img src="https://latex.codecogs.com/gif.latex?S_1 = \alpha^2"/>
- For the second syndrome, <img src="https://latex.codecogs.com/gif.latex?m_2 = m_1"/>, so <img src="https://latex.codecogs.com/gif.latex?S_2 = b_1(\alpha^2) = (\alpha^2)^2"/>
	- <img src="https://latex.codecogs.com/gif.latex?S_2 = \alpha^4"/>
- For the third syndrome, <img src="https://latex.codecogs.com/gif.latex?m_3 = x^4 + x^3 + x^2 + x + 1. b_3 = (x^8+1)"/> mod <img src="https://latex.codecogs.com/gif.latex?(x^4+x^3+x^2+x+1) = x^3+1"/>. <img src="https://latex.codecogs.com/gif.latex?S_3 = b_3(\alpha^3) = \alpha^9 + 1 = z^3+z+1 = \alpha^7"/>
	- <img src="https://latex.codecogs.com/gif.latex?S_3 = \alpha^7"/>
- For the fourth syndrome, <img src="https://latex.codecogs.com/gif.latex?m_4 = m_1"/>, so <img src="https://latex.codecogs.com/gif.latex?S_4 = (\alpha^4)^2 = \alpha^8"/>
	- <img src="https://latex.codecogs.com/gif.latex?S_4 = \alpha^8"/>

So <img src="https://latex.codecogs.com/gif.latex?(S_1, S_2, S_3, S_4) = (\alpha^2, \alpha^4, \alpha^7, \alpha^8)"/>

Since <img src="https://latex.codecogs.com/gif.latex?S(\alpha^i) = e(\alpha^i)"/>, the syndromes <img src="https://latex.codecogs.com/gif.latex?S_1, S_2,...,S_{2t}"/> can also be written

<img src="https://latex.codecogs.com/gif.latex?S_1 = (\alpha^1)^{j_1} + (\alpha^1)^{j_2} + ... + (\alpha^1)^{j_{\nu}}"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?S_2 = (\alpha^2)^{j_1} + (\alpha^2)^{j_2} + ... + (\alpha^2)^{j_{\nu}}"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?..."/> <br/>
<img src="https://latex.codecogs.com/gif.latex?S_{2t} = (\alpha^{2t})^{j_1} + (\alpha^{2t})^{j_2} + ... + (\alpha^{2t})^{j_{\nu}}"/> <br/>

or alternatively,

<img src="https://latex.codecogs.com/gif.latex?S_1 = (\alpha^{j_1})^1 + (\alpha^{j_2})^1 + ... + (\alpha^{j_{\nu}})^1"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?S_2 = (\alpha^{j_1})^2 + (\alpha^{j_2})^2 + ... + (\alpha^{j_{\nu}})^2"/> <br/>
<img src="https://latex.codecogs.com/gif.latex?..."/> <br/>
<img src="https://latex.codecogs.com/gif.latex?S_{2t} = (\alpha^{j_1})^{2t} + (\alpha^{j_2})^{2t} + ... + (\alpha^{j_{\nu}})^{2t}"/> <br/>

Solving for the unknowns <img src="https://latex.codecogs.com/gif.latex?\{\alpha^{j_1}, \alpha^{j_2}, ..., \alpha^{j_{\nu}}\}"/> is the goal of any algorithm for
decoding BCH codes. However, there are in general <img src="https://latex.codecogs.com/gif.latex?2^k"/> possible solutions, so, if the number of errors <img src="https://latex.codecogs.com/gif.latex?\nu"/> is
less than the designed error correcting capability t, then the solution which yields the smallest number of
errors is the correct solution. This corresponds to the solution with the smallest <img src="https://latex.codecogs.com/gif.latex?\nu"/>.

The syndrome-error equations can be simplified by defining

<img src="https://latex.codecogs.com/gif.latex?\beta_i = \alpha^{j_i}"/>

after which the syndrome-error equations can be written

<img src="https://latex.codecogs.com/gif.latex?S_1 = \beta_1 + \beta_2 + ... + \beta_{\nu}"/><br/>
<img src="https://latex.codecogs.com/gif.latex?S_2 = \beta_1^2 + \beta_2^2 + ... + \beta_{\nu}^2"/><br/>
<img src="https://latex.codecogs.com/gif.latex?..."/><br/>
<img src="https://latex.codecogs.com/gif.latex?S_{2t} = \beta_1^{2t} + \beta_2^{2t} + ... + \beta_{\nu}^{2t}"/><br/>

These are power sum symmetric functions. Using the <img src="https://latex.codecogs.com/gif.latex?\beta"/>'s, we can define the error locator polynomial <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/> as

<img src="https://latex.codecogs.com/gif.latex?\sigma(x) = (1 + \beta_1 x)(1 + \beta_2 x) ... (1 + \beta_{\nu} x)"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= \sigma_0 + \sigma_1 x + \sigma_2 x^2 + ... + \sigma_{\nu} x^\nu"/><br/>

The roots of <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/> are <img src="https://latex.codecogs.com/gif.latex?\beta_1^{-1}"/>, <img src="https://latex.codecogs.com/gif.latex?\beta_2^{-1}"/>, ..., <img src="https://latex.codecogs.com/gif.latex?\beta_{\nu}^{-1}"/>, which are the inverses of the
error locations. This leap of defining an additional polynomial might seem arbitrary, but it's for a good
reason. If <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/> is expanded out, the relationship between its coefficients and the <img src="https://latex.codecogs.com/gif.latex?\beta"/>'s are in the form of elementary symmetric polynomials:

<img src="https://latex.codecogs.com/gif.latex?\sigma_0 = 1"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\sigma_1 = \beta_1 + \beta_2 + ... + \beta_{\nu}"/><br/>
<img src="https://latex.codecogs.com/gif.latex?\sigma_2 = \beta_1\beta_2 + \beta_2\beta_3 + ... + \beta_{\nu-1}\beta_{\nu}"/><br/>
<img src="https://latex.codecogs.com/gif.latex?..."/><br/>
<img src="https://latex.codecogs.com/gif.latex?\sigma_{\nu} = \beta_1\beta_2...\beta_{\nu}"/><br/>

From the theory of elementary symmetric polynomials, we know that the syndromes and the coefficients of <img src="https://latex.codecogs.com/gif.latex?\sigma"/> must obey Newton's identities:

<img src="https://latex.codecogs.com/gif.latex?S_1 + \sigma_1 = 0"/><br/>
<img src="https://latex.codecogs.com/gif.latex?S_2 + \sigma_1 S_1 + 2\sigma_2 = 0"/><br/>
<img src="https://latex.codecogs.com/gif.latex?S_3 + \sigma_1 S_2 + \sigma_2 S_1 + 3\sigma_3 = 0"/><br/>
<img src="https://latex.codecogs.com/gif.latex?..."/><br/>
<img src="https://latex.codecogs.com/gif.latex?S_{\nu} + \sigma_1 S_{\nu-1} + ... + \sigma_{\nu-1} S_1 + \nu \sigma_{\nu} = 0"/><br/>
<img src="https://latex.codecogs.com/gif.latex?S_{\nu+1} + \sigma_1 S_{\nu} + ... + \sigma_{\nu-1} S_2 + \sigma_{\nu} S_1 = 0"/><br/>
<img src="https://latex.codecogs.com/gif.latex?..."/><br/>

So defining <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/> in this way links the error location values <img src="https://latex.codecogs.com/gif.latex?\beta_i"/> with the syndromes <img src="https://latex.codecogs.com/gif.latex?S_i"/> via Newton's
identities. Our objective is to determine the error locator polynomial's coefficients, after which, the roots of
the error locator polynomial can be solved for. There may be many such <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/>'s, but we want the one with
minimal degree. The process for decoding the error has three steps:

1. Compute the syndromes <img src="https://latex.codecogs.com/gif.latex?S_i"/> by evaluating <img src="https://latex.codecogs.com/gif.latex?r(x)"/> at <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> for <img src="https://latex.codecogs.com/gif.latex?i \in \{1,2,...,2t\}"/>
2. Using the syndromes, compute the error locator polynomial <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/>, whose roots are the inverse of the
locations of the errors
3. For binary codes, once the error locations are known, those bits just need to be flipped in the code. A practical consideration is that bits are indexed from 1 to n instead of 0 to n-1, so an error at <img src="https://latex.codecogs.com/gif.latex?\alpha^1"/> is an error at the least significant bit, and an error at <img src="https://latex.codecogs.com/gif.latex?\alpha^{15}=1"/> is an error at the most significant bit.

In this section, we'll cover one particular algorithm for decoding BCH codes. Berlekamp's algorithm gives an iterative procedure for solving for the <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/> with smallest degree that satisfies Newton's identities. The algorithm does this by constructing trial functions <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu)}"/> which
satisfy the first <img src="https://latex.codecogs.com/gif.latex?\mu"/> Newton's identities one by one until <img src="https://latex.codecogs.com/gif.latex?\sigma^{(2t)}"/> is reached, which is minimal degree
polynomial that satisfies the first 2t Newton's identities. Once <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/> is known, all that remains to be done
is to find its roots by exhaustively substituting all n-1 values of <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> to see which ones result in
<img src="https://latex.codecogs.com/gif.latex?\sigma(\alpha^i) = 0"/>. The error locations are then the inverses of those <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/>'s. Remember, in a finite field
the inverse of <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> is simply <img src="https://latex.codecogs.com/gif.latex?\alpha^{n-i}"/>, where n+1 is the order of the field.

Let's go into detail about the procedure. First, let

<img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu)} = 1 + \sigma_1^{(\mu)} x + \sigma_2^{(\mu)} x^2 + ... + \sigma_{l_{\mu}}^{(\mu)} x^{l_{\mu}}"/>

be the <img src="https://latex.codecogs.com/gif.latex?\sigma"/> constructed on the <img src="https://latex.codecogs.com/gif.latex?\mu"/>th iterative step, whose coefficients produce a polynomial that satisfies
the first <img src="https://latex.codecogs.com/gif.latex?\mu"/> Newton's identifies. <img src="https://latex.codecogs.com/gif.latex?l_{\mu}"/> is the degree of <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu)}"/>. To determine <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu+1)}"/>, we
must make a correction to <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu)}"/> to make sure it satisfies the first <img src="https://latex.codecogs.com/gif.latex?\mu + 1"/> Newton's identities. We do this by computing the discrepancy:

<img src="https://latex.codecogs.com/gif.latex?d_{\mu} = S_{\mu+1} + \sigma_1^{(\mu)} S_{\mu} + \sigma_2^{(\mu)} S_{\mu-1} + ... + \sigma_{l_{\mu}}^{(\mu)} S_{u+1-l_{\mu}}"/>

if <img src="https://latex.codecogs.com/gif.latex?d_{\mu} = 0"/>, we simply set <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu+1)} = \sigma^{(\mu)}"/>. If <img src="https://latex.codecogs.com/gif.latex?d_{\mu}"/> is nonzero, we need to look to a
previous iteration, indexed at <img src="https://latex.codecogs.com/gif.latex?\rho"/>, where <img src="https://latex.codecogs.com/gif.latex?d_{\rho} \neq 0"/> and <img src="https://latex.codecogs.com/gif.latex?\rho - l_{\rho}"/> is largest. With the <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\rho)}"/>
from that iteration, set

<img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu + 1)} = \sigma^{\mu} + d_{\mu}d_{\rho}^{-1}x^{\mu-\rho}\sigma^{(\rho)}"/>

which gives the minimal polynomial which satisfies the first <img src="https://latex.codecogs.com/gif.latex?\mu+1"/> Newton's identities. This step will definitely be confusing for any reasonable person, but hopefully it will make sense when we do an example. Also, so far the whole procedure has been presented abstractly, but the thing to remember is that everything, from the coefficients of <img src="https://latex.codecogs.com/gif.latex?\sigma"/>
to the discrepancies d to the error locator values <img src="https://latex.codecogs.com/gif.latex?\beta"/>, are all elements of the finite field. So they're all
an <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> (including 1) or 0.

The Berlekamp procedure is best summarized with a table. Regardless of the problem, the default starting table is
is:

| <img src="https://latex.codecogs.com/gif.latex?\mu"/> | <img src="https://latex.codecogs.com/gif.latex?S_{\mu}"/> | <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu)}"/> | <img src="https://latex.codecogs.com/gif.latex?d_\mu"/> | <img src="https://latex.codecogs.com/gif.latex?l_{\mu}"/> | <img src="https://latex.codecogs.com/gif.latex?\mu-l_{\mu}"/> |
|-----|---------|----------------|---------|---------|-------------|
| -1  | -       | 1              | 1       | 0       | -1          |
| 0   | -       | 1              | <img src="https://latex.codecogs.com/gif.latex?S_1"/>     | 0       | 0           |

Rows are filled in one at a time until row 2t, and we start on row 0. Assuming we've filled up to the <img src="https://latex.codecogs.com/gif.latex?\mu"/>th row, filling the <img src="https://latex.codecogs.com/gif.latex?\mu+1"/>st row is done as follows:

1. Compute <img src="https://latex.codecogs.com/gif.latex?d_{\mu}"/> for the row
2. If <img src="https://latex.codecogs.com/gif.latex?d_{\mu} = 0"/>, then <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu+1)} = \sigma^{(\mu)}"/> and <img src="https://latex.codecogs.com/gif.latex?l_{\mu+1} = l_{\mu}"/>
3. If <img src="https://latex.codecogs.com/gif.latex?d_{\mu} \neq 0"/>, then find another row prior to (but not including) the <img src="https://latex.codecogs.com/gif.latex?\mu"/> th row, row <img src="https://latex.codecogs.com/gif.latex?\rho"/>, such that
<img src="https://latex.codecogs.com/gif.latex?d_{\rho} \neq 0"/> and <img src="https://latex.codecogs.com/gif.latex?\rho - l_{\rho}"/> is the largest value. Then update <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu+1)}"/> via

	<img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu+1)} = \sigma^{(\mu)} + d_{\mu}d_{\rho}^{-1}x^{\mu-\rho}\sigma^{(\rho)}"/>

	and <img src="https://latex.codecogs.com/gif.latex?l_{\mu+1} = max(l_{\mu}, l_{\rho} + \mu - \rho)"/> -- but this equation isn't needed in practice though: <img src="https://latex.codecogs.com/gif.latex?l_{\mu}"/> is always the degree of <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu)}"/>, which can be easily read off.

All of this sounds super complicated, but it really isn't: it's just tedious. Let's do an example to see how it
works in practice. Let's say that we are using a (15,5) triple error correcting code, and the transmitted
codeword v = 000000000000000 is received as r = 001000000101000, or <img src="https://latex.codecogs.com/gif.latex?r(x)=x^{12}+x^5+x^3"/>. The first step is to
compute the syndromes. There are six syndromes to compute since we are using a triple error correcting code. To follow the computations, the reader may find it useful to refer to the tables for <img src="https://latex.codecogs.com/gif.latex?GF(16)"/> that we compiled in an earlier section.

We need to compute syndromes for <img src="https://latex.codecogs.com/gif.latex?\alpha^1"/> to <img src="https://latex.codecogs.com/gif.latex?\alpha^6"/>. The minimal polynomial for <img src="https://latex.codecogs.com/gif.latex?\alpha^1"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^2"/>, and
<img src="https://latex.codecogs.com/gif.latex?\alpha^4"/> is <img src="https://latex.codecogs.com/gif.latex?m_1(x)=x^4+x+1"/>. The minimal polynomial for <img src="https://latex.codecogs.com/gif.latex?\alpha^3"/> and <img src="https://latex.codecogs.com/gif.latex?\alpha^6"/> is <img src="https://latex.codecogs.com/gif.latex?m_3(x)=x^4+x^3+x^2+x+1"/>. The
minimal polynomial for <img src="https://latex.codecogs.com/gif.latex?\alpha^5"/> is <img src="https://latex.codecogs.com/gif.latex?m_5(x)=x^2+x+1"/>. By pen and paper, the easiest way to compute the syndromes is
to compute the remainder of <img src="https://latex.codecogs.com/gif.latex?r(x)"/> after dividing by <img src="https://latex.codecogs.com/gif.latex?m_i(x)"/>, and evaluating the remainder <img src="https://latex.codecogs.com/gif.latex?b_i(x)"/> at <img src="https://latex.codecogs.com/gif.latex?\alpha_i"/>. Let's  do this for each <img src="https://latex.codecogs.com/gif.latex?b_i(x)m_i(x)"/>:

- <img src="https://latex.codecogs.com/gif.latex?b_1 = (x^{12}+x^5+x^3) \text{ mod } (x^4+x+1) = 1"/>
- <img src="https://latex.codecogs.com/gif.latex?b_3 = (x^{12}+x^5+x^3) \text{ mod } (x^4+x^3+x^2+x+1) = x^3+x^2+1"/>
- <img src="https://latex.codecogs.com/gif.latex?b_5 = (x^{12}+x^5+x^3) \text{ mod } (x^2+x+1) = x^2"/>

So:

- <img src="https://latex.codecogs.com/gif.latex?S_1 = b_1(\alpha) = 1"/>
- <img src="https://latex.codecogs.com/gif.latex?S_2 = b_1(\alpha^2) = 1"/>
- <img src="https://latex.codecogs.com/gif.latex?S_3 = b_3(\alpha^3) = \alpha^9 + \alpha^6 + 1 = z^3+z + z^3+z^2 + 1 = z^2+z+1 = \alpha^10"/>
- <img src="https://latex.codecogs.com/gif.latex?S_4 = b_1(\alpha^4) = 1"/>
- <img src="https://latex.codecogs.com/gif.latex?S_5 = b_5(\alpha^5) = (\alpha^5)^2 = \alpha^{10}"/>
- <img src="https://latex.codecogs.com/gif.latex?S_6 = b_3(\alpha^6) = \alpha^{18} + \alpha^{12} + 1 = \alpha^{12} + \alpha^3 + 1 = z^3+z^2+z+1+z^3+1 = z^2+z+1 = \alpha^{5}"/>

We'll write down the final table first, and then dissect the results row by row:

| <img src="https://latex.codecogs.com/gif.latex?\mu"/> | <img src="https://latex.codecogs.com/gif.latex?S_{\mu}"/>     | <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu)}"/>   | <img src="https://latex.codecogs.com/gif.latex?d_{\mu}"/>     | <img src="https://latex.codecogs.com/gif.latex?l_{\mu}"/> | <img src="https://latex.codecogs.com/gif.latex?\mu-l_{\mu}"/> |
|-----|-------------|------------------|-------------|---------|-------------|
| -1  | -           | 1                | 1           | 0       | -1          |
| 0   | -           | 1                | <img src="https://latex.codecogs.com/gif.latex?S_1"/>         | 0       | 0           |
| 1   | 1           | <img src="https://latex.codecogs.com/gif.latex?x+1"/>              | 0           | 1       | 0           |
| 2   | 1           | <img src="https://latex.codecogs.com/gif.latex?x+1"/>              | <img src="https://latex.codecogs.com/gif.latex?\alpha^5"/>    | 1       | 1           |
| 3   | <img src="https://latex.codecogs.com/gif.latex?\alpha^{10}"/> | <img src="https://latex.codecogs.com/gif.latex?\alpha^5 x^2+x+1"/> | 0           | 2       | 1           |
| 4   | 1           | <img src="https://latex.codecogs.com/gif.latex?\alpha^5 x^2+x+1"/> | <img src="https://latex.codecogs.com/gif.latex?\alpha^{10}"/> | 2       | 2           |
| 5   | <img src="https://latex.codecogs.com/gif.latex?\alpha^{10}"/> | <img src="https://latex.codecogs.com/gif.latex?\alpha^5 x^3+x+1"/> | 0           | 3       | 2           |
| 6   | <img src="https://latex.codecogs.com/gif.latex?\alpha^5"/>    | <img src="https://latex.codecogs.com/gif.latex?\alpha^5 x^3+x+1"/> | -           | -       | -           |

We start at <img src="https://latex.codecogs.com/gif.latex?\mu=0"/>. Since our discrepancy (by default) is <img src="https://latex.codecogs.com/gif.latex?S_1=1"/>, which is not zero, to compute <img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu+1)}"/>,
we need to choose a previous row <img src="https://latex.codecogs.com/gif.latex?\rho"/> where <img src="https://latex.codecogs.com/gif.latex?d_{\rho} \neq 0"/>, with the largest <img src="https://latex.codecogs.com/gif.latex?\rho-l_{\rho}"/>. Since we are
not allowed to choose our current row, there is only one choice: <img src="https://latex.codecogs.com/gif.latex?\rho=-1"/>. Using this row, we have

<img src="https://latex.codecogs.com/gif.latex?\sigma^{(1)} = \sigma^{(0)} + d_{0}d_{-1}^{-1}x^{0-(-1)}\sigma^{(-1)}"/>
<img src="https://latex.codecogs.com/gif.latex?= 1 + (1)(1)x^{0-(-1)}(1)"/>
<img src="https://latex.codecogs.com/gif.latex?= x + 1"/>

This allows us to move to the <img src="https://latex.codecogs.com/gif.latex?\mu=1"/> row. At this row, <img src="https://latex.codecogs.com/gif.latex?l_{1} = 1"/> since the degree of <img src="https://latex.codecogs.com/gif.latex?\sigma^{(1)}"/> is 1, and so
<img src="https://latex.codecogs.com/gif.latex?1-l_{1} = 1-1 = 0"/>. Now we need to compute the discrepancy for this row, which is done by computing

<img src="https://latex.codecogs.com/gif.latex?d_{\mu} = S_{\mu+1} + \sigma_1^{(\mu)} S_{\mu} + \sigma_2^{(\mu)} S_{\mu-1} + ... + \sigma_{l_{\mu}}^{(\mu)} S_{u+1-l_{\mu}}"/>

Here, <img src="https://latex.codecogs.com/gif.latex?\mu=1"/>, <img src="https://latex.codecogs.com/gif.latex?S_{\mu+1} = S_2 = 1"/>, <img src="https://latex.codecogs.com/gif.latex?\sigma_1^{(\mu)}=1"/>, and <img src="https://latex.codecogs.com/gif.latex?S_{\mu}=S_1=1"/>, so <img src="https://latex.codecogs.com/gif.latex?d_1 = 1 + (1)(1) = 1 + 1 = 0"/>. Since
the discrepancy is zero, we're free to move onto the next row by setting

<img src="https://latex.codecogs.com/gif.latex?\sigma^{(\mu+1)} = \sigma^{(\mu)}"/>

so <img src="https://latex.codecogs.com/gif.latex?\sigma^{(2)} = \sigma^{(1)} = x+1"/>, and <img src="https://latex.codecogs.com/gif.latex?l_2 = 1"/>, and <img src="https://latex.codecogs.com/gif.latex?2-l_2 = 2-1 = 1"/>. Again, we need to compute the
discrepancy:

<img src="https://latex.codecogs.com/gif.latex?d_2 = S_3 + (1)(S_2) = \alpha^{10} + 1 = z^2+z+1+1 = z^2+z = \alpha^5"/>

Since <img src="https://latex.codecogs.com/gif.latex?d_2"/> is not zero, we need to select the previous column with largest <img src="https://latex.codecogs.com/gif.latex?\rho-l_{\rho}"/>. We have two choices this time 
where <img src="https://latex.codecogs.com/gif.latex?d_{\rho} \neq 0"/>: <img src="https://latex.codecogs.com/gif.latex?\rho=-1"/> and <img src="https://latex.codecogs.com/gif.latex?\rho=0"/>, of which <img src="https://latex.codecogs.com/gif.latex?\rho=0"/> has the largest <img src="https://latex.codecogs.com/gif.latex?\rho-l_{\rho}"/>. Using this
row, we are able to compute <img src="https://latex.codecogs.com/gif.latex?\sigma^{(3)}"/>:

<img src="https://latex.codecogs.com/gif.latex?\sigma^{(3)} = \sigma^{(2)} + d_2 d_0^{-1} x^{2-0} \sigma^{(0)}"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= (x+1) + \alpha^5 x^2 (1)"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= \alpha^5 x^2 + x + 1"/><br/>

On this row, <img src="https://latex.codecogs.com/gif.latex?l_3 = 2"/> and <img src="https://latex.codecogs.com/gif.latex?3-l_3 = 3-2 = 1"/>. The discrepancy is

<img src="https://latex.codecogs.com/gif.latex?d_3 = S_4 + (1)(S_3) + \alpha^5 S_2 = 1 + \alpha^{10} + \alpha^5 = 1 + z^2+z+1 +z^2+z = 0"/>

Since <img src="https://latex.codecogs.com/gif.latex?d_3=0"/>, we are free to move onto <img src="https://latex.codecogs.com/gif.latex?\mu=4"/> with the same <img src="https://latex.codecogs.com/gif.latex?\sigma"/>:

<img src="https://latex.codecogs.com/gif.latex?\sigma^{(4)} = \sigma^{(3)} = \alpha^5 x^2 + x + 1"/>

On this row, <img src="https://latex.codecogs.com/gif.latex?l_4 = 2"/> and <img src="https://latex.codecogs.com/gif.latex?4-l_4 = 2"/>. The discrepancy is

<img src="https://latex.codecogs.com/gif.latex?d_4 = S_5 + (1)(S_4) + \alpha^5 S_3 = \alpha^{10} + 1 + 1 = \alpha^{10}"/>

Since <img src="https://latex.codecogs.com/gif.latex?d_4"/> is nonzero, we have to again choose a previous row. The options available with <img src="https://latex.codecogs.com/gif.latex?d_{\rho} \neq 0"/> are <img src="https://latex.codecogs.com/gif.latex?\rho=-1"/>, <img src="https://latex.codecogs.com/gif.latex?\rho=0"/>, and <img src="https://latex.codecogs.com/gif.latex?\rho=2"/>. Of these, <img src="https://latex.codecogs.com/gif.latex?\rho=2"/> has the largest <img src="https://latex.codecogs.com/gif.latex?\rho-l_{\rho}"/> of 1. With this <img src="https://latex.codecogs.com/gif.latex?\rho"/>,

<img src="https://latex.codecogs.com/gif.latex?\sigma^{(5)} = \sigma^{(4)} + d_4 d_2^{-1} x^{4-2} \sigma^{(2)}"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= \alpha^5 x^2 + x + 1 + \alpha^{10} \alpha^{10} x^2 (x+1)"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= \alpha^5 x^2+x+1+\alpha^5 x^3+\alpha^5 x^2"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= \alpha^5 x^3 + x + 1"/><br/>
(where we used the fact that <img src="https://latex.codecogs.com/gif.latex?(\alpha^5)^{-1} = \alpha^{15-5} = \alpha^{10}"/> and that <img src="https://latex.codecogs.com/gif.latex?\alpha^{10}\alpha^{10} = \alpha^{15}\alpha^{5} = \alpha^{5}"/>)

At row 5, <img src="https://latex.codecogs.com/gif.latex?l_{5}=3"/> and <img src="https://latex.codecogs.com/gif.latex?5-l_5= 5-3 = 2"/> . The discrepancy at this row is

<img src="https://latex.codecogs.com/gif.latex?d_5 = S_6 + (1)(S_5) + (0)(S_4) + \alpha^5 S_3"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= \alpha^5 + \alpha^{10} + \alpha^{15}"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= \alpha^{10} + \alpha^5 + 1"/><br/>
<img src="https://latex.codecogs.com/gif.latex?= z^2+z+1+z^2+z+1= 0"/><br/>

Since <img src="https://latex.codecogs.com/gif.latex?d_5 = 0"/>, <img src="https://latex.codecogs.com/gif.latex?\sigma^{(6)} = \sigma^{(5)} = \alpha^5 x^3 + x + 1"/>. Furthermore, since we've reached row 6, which is the final row, we have found <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/>. So the error locator polynomial is

<img src="https://latex.codecogs.com/gif.latex?\sigma(x) = \alpha^5 x^3 + x + 1"/>

Now all that remains is to find the roots of <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/>. We can do this by exhaustively trying all values of
<img src="https://latex.codecogs.com/gif.latex?\alpha^i"/>. We will do the first few to get a taste of the arithmetic, and then state the other results.

- <img src="https://latex.codecogs.com/gif.latex?\sigma(1) = \alpha^5"/>
- <img src="https://latex.codecogs.com/gif.latex?\sigma(\alpha) = \alpha^6 + \alpha + 1 = z^3+z^2+z+1 = \alpha^{12}"/>
- <img src="https://latex.codecogs.com/gif.latex?\sigma(\alpha^2) = \alpha^{11} + \alpha^2 + 1 = z^3+z^2+z+z^2+1 = \alpha^{10}"/>
- <img src="https://latex.codecogs.com/gif.latex?\sigma(\alpha^3) = \alpha^{14} + \alpha^2 + 1 = z^3+1+z^3+1 = 0"/>

Similarly, <img src="https://latex.codecogs.com/gif.latex?\sigma(\alpha^{10}) = 0"/>, and <img src="https://latex.codecogs.com/gif.latex?\sigma(\alpha^{12}) = 0"/>, so <img src="https://latex.codecogs.com/gif.latex?\alpha^3"/>, <img src="https://latex.codecogs.com/gif.latex?\alpha^{10}"/>, and <img src="https://latex.codecogs.com/gif.latex?\alpha^{12}"/> are the roots
of the error locator polynomial. Note that the Galois field element 1 corresponds to <img src="https://latex.codecogs.com/gif.latex?\alpha^{15}"/>. To the the error locations, we take the inverses of each of these:

- <img src="https://latex.codecogs.com/gif.latex?(\alpha^3)^{-1} = \alpha^{15-3} = \alpha^{12}"/>
- <img src="https://latex.codecogs.com/gif.latex?(\alpha^{10})^{-1} = \alpha^{15-10} = \alpha^5"/>
- <img src="https://latex.codecogs.com/gif.latex?(\alpha^{12})^{-1} = \alpha^{15-12} = \alpha^3"/>

So the error locations are at bits 3, 5, and 12, which is where we put the errors in the beginning of the example. Again, note that if one of the roots of <img src="https://latex.codecogs.com/gif.latex?\sigma(x)"/> had been 1, then the inverse would also be 1, which indicates an error on the 15th, or most significant bit, since bits are indexed from 1 to 15 rather than from 0 to 14.