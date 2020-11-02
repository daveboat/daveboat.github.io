---
layout: post
title: An introduction to BCH codes
date: 2020-11-02 09:00
summary: An introduction to Bose–Chaudhuri–Hocquenghem (BCH) cyclic error correcting codes, aimed at engineers who want to understand the background without all of the mathematical rigor.
categories: blogs
---

I've been working with error correcting codes, and I had a hard time finding good, consolidated sources online which explained the background at a level of detail that I was comfortable with. Things were either very high level, which didn't give enough details to implement a real solution, or overly theoretical, which doesn't appeal to anyone not accustomed to the theorem, proof, lemma, proof style of graduate math textbooks. What I hope to do here is give an explanation of cyclic error correcting codes and BCH codes, aimed at engineers and applied scientists who want to understand an implementation and get a flavor of the theory, without going into proofs. For this reason, I'll just state results without proving them. Instead, I will try to give an outline of how to implement each part of the process. Those interested in more complete picture are encouraged refer to [Costello and Lin](https://books.google.ca/books/about/Error_Control_Coding.html?id=autQAAAAMAAJ&redir_esc=y).

This blog will be divided into the following sections:
1. Background on finite fields
2. Introduction to binary cyclic linear error correcting codes
3. Introduction to BCH codes
4. Encoding BCH codes
5. A sketch of how to decode BCH codes

## Finite fields

The mathematics of cyclic error correcting codes is based on finite fields, so it's impossible to have a meaningful discussion of cyclic codes without first giving the necessary background on finite fields. We'll try to cover the minimum necessary material to understand future sections, with examples along the way to aid understanding. Let's get started!

**Finite fields**, or **Galois fields**, written <img src="https://latex.codecogs.com/gif.latex?GF(q)"/>, are fields (i.e. they are closed under, and have inverses for, addition and multiplication) which have a finite number of elements. The simplest examples are the integers modulo a prime -- <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> is the field <img src="https://latex.codecogs.com/gif.latex?\{0,1\}"/> with addition and multiplication modulo 2, <img src="https://latex.codecogs.com/gif.latex?GF(3)"/> is the field <img src="https://latex.codecogs.com/gif.latex?\{0, 1, 2\}"/> with addition and multiplication modulo 3, etc. To aid understanding, let's write out the addition and multiplication table for <img src="https://latex.codecogs.com/gif.latex?GF(5)"/>. When adding or multiplying, everything is modulo 5, so 4 + 4 = 8 mod 5 = 3, and 4 * 3 = 12 mod 5 = 2 for example.

The addition table is:

| x  | 0 | 1 | 2 | 3 | 4 |
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

Galois fields must have a number of elements (order) equal to a prime power, and all Galois fields with the same order are isomorphic to each other. To construct a Galois field with non-prime number of elements (i.e. a prime power number of elements), or an **extension field**, we use polynomials instead of integers to represent field elements. To construct <img src="https://latex.codecogs.com/gif.latex?GF(16) = GF(2^4)"/>, we use <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/P"/>. <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]"/> is pronounced "GF(2) adjoin x", and represents the set of all polynomials whose coefficients are in <img src="https://latex.codecogs.com/gif.latex?GF(2)"/>. <img src="https://latex.codecogs.com/gif.latex?P"/>
is an **irreducible polynomial** of degree 2 over <img src="https://latex.codecogs.com/gif.latex?GF(2)"/>. Irreducible polynomials are polynomials which cannot be factored. For example, in <img src="https://latex.codecogs.com/gif.latex?GF(2)"/>, <img src="https://latex.codecogs.com/gif.latex?x^2+x+1"/> is irreducible, but <img src="https://latex.codecogs.com/gif.latex?x^2+1"/> is not, since <img src="https://latex.codecogs.com/gif.latex?(x+1)(x+1)=x^2+2x+1"/>, but 2=0, so <img src="https://latex.codecogs.com/gif.latex?(x+1)(x+1)=x^2+1"/>. The irreducible polynomials of the first few orders for <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> are:

| Order | Irreducible Polynomial                                                                     |
|-------|--------------------------------------------------------------------------------------------|
| 2     | <img src="https://latex.codecogs.com/gif.latex?x^2+x+1"/>                                                                                   |
| 3     | <img src="https://latex.codecogs.com/gif.latex?x^3+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^3+x^2+1  "/>                                                                       |
| 4     | <img src="https://latex.codecogs.com/gif.latex?x^4+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^4+x^3+x^2+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^4+x^3+1 "/>                                                       |
| 5     | <img src="https://latex.codecogs.com/gif.latex?x^5+x^2+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^3+x^2+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^3+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^4+x^3+x+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^4+x^3+x^2+1"/>, <img src="https://latex.codecogs.com/gif.latex?x^5+x^4+x^2+x+1"/> |

There are, in general, multiple irreducible polynomials of each degree for each base field. In practice, we choose the
one that has the smallest overall degree (so we prefer to use <img src="https://latex.codecogs.com/gif.latex?x^3+x+1"/> instead of <img src="https://latex.codecogs.com/gif.latex?x^3+x^2+1  "/>, for example). Since all finite fields of an order are isomorphic, constructing an extension field of a particular order with any valid irreducible polynomial creates an isomorphic finite field (the addition and multiplication tables will just have different entries). Let's do an example of constructing an extension field <img src="https://latex.codecogs.com/gif.latex?GF(9)"/> from the base field <img src="https://latex.codecogs.com/gif.latex?GF(3)"/>. We choose <img src="https://latex.codecogs.com/gif.latex?P = x^2 + 1"/>. <img src="https://latex.codecogs.com/gif.latex?GF(3)[x]/(x^2+1)"/> is then the set of all polynomials with coefficients in <img src="https://latex.codecogs.com/gif.latex?GF(3)"/>, modulo <img src="https://latex.codecogs.com/gif.latex?x^2+1"/>, or the set <img src="https://latex.codecogs.com/gif.latex?\{0, 1, 2, x, x+1, x+2, 2x, 2x+1, 2x+2\}"/>, which has 9 elements. Addition and multiplication using this polynomial representation is modulo <img src="https://latex.codecogs.com/gif.latex?x^2+1"/> and modulo 3. Again, for understanding, let's write out the multiplication table (the addition table is easier to calculate) for <img src="https://latex.codecogs.com/gif.latex?GF(3)[x]/(x^2+1)"/> and work out some specific examples.

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

To someone who has done regular arithmetic their whole life, polynomial modular arithmetic can seem strange and unwieldy at first, but you may find it more enjoyable once you get the hang of it! 

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

We've been using the base field <img src="https://latex.codecogs.com/gif.latex?GF(3)"/> in this section to give an example with an order higher than 2. In the remainder of this blog though, we'll use <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> as the base field in examples, since ultimately we're interested in binary codes, for which <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> is always the base field.

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

- **Linear**: any codeword + another codeword is a valid codeword

- **Binary**: only 1's and 0's in the codewords

- **Cyclic**: any cyclic shift of a codeword is another codeword

For example, all sets of valid 4-length linear binary cyclic codes are:

- {0000, 1111}
- {0000, 0101, 1010, 1111}
- {0000, 0110, 0011, 1100, 0001, 0010, 0100, 1000, 0101, 1010, 1001, 1111, 0111, 1110, 1011, 1101}

Binary linear cyclic codes are represented by <img src="https://latex.codecogs.com/gif.latex?GF(2)[x]/(x^n-1)"/>. In this representation, multiplying by <img src="https://latex.codecogs.com/gif.latex?x"/> amounts to
a left-shift by 1, since <img src="https://latex.codecogs.com/gif.latex?x^n=1"/>. Extension fields of <img src="https://latex.codecogs.com/gif.latex?GF(2)"/> naturally work to represent sequences of binary numbers, since their coefficients are either 0 or 1, so the resulting polynomials can be thought of as binary messages with the most significant bit (MSB) at either the highest degree of <img src="https://latex.codecogs.com/gif.latex?x"/> or lowest degree of <img src="https://latex.codecogs.com/gif.latex?x"/>.

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

## A sketch of decoding BCH codes

Decoding is a more complex process worth its own blog. I might go over BCH decoding in detail in a subsequent blog, but I'll only provide a sketch here. 

Decoding BCH codes has the following steps:
1. Calculate the syndromes by evaluating the received polynomial at <img src="https://latex.codecogs.com/gif.latex?\alpha^i"/> for i=1 to 2t
2. If there are any non-zero syndromes, then there are errors in the received codeword, otherwise go to step 5
3. The syndromes are used to compute the error locator polynomial <img src="https://latex.codecogs.com/gif.latex?\Lambda(x)"/>. This can be done using the iterative
Berlekamp-Massey algorithm or the Peterson–Gorenstein–Zierler (PGZ) algoritm. The error locator polynomial is of the form

    <img src="https://latex.codecogs.com/gif.latex?\Lambda(x)=1 + \lambda_1 x + \lambda_2 x^2 + ... + \lambda_t x^t"/>

    where t is the number of bit errors in the codeword. Computing the error locator polynomial is the most complicated and expensive step.
4. The roots of the error locator polynomial give the locations of the errors (as inverses). Since we are
dealing with binary codewords, the bits at those locations just need to be flipped. The modular polynomial
factorization can be done by brute force by just trying all <img src="https://latex.codecogs.com/gif.latex?\alpha...\alpha^{n-1}"/> possibilities via substitution.
5. After error correction, the original message can just be read as the k most significant bits of the codeword if the codeword was encoded systematically, or recovered by dividing by <img src="https://latex.codecogs.com/gif.latex?g(x)"/> if it was encoded non-systematically.

There is an example of decoding in the Wikipedia article [here](https://en.wikipedia.org/wiki/BCH_code#Decoding), and in Section 6.2 of Costello and Lin.

