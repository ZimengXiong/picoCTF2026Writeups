# ClusterRSA

`ClusterRSA` is a cryptography challenge that looks like ordinary RSA at first glance because the artifact only gives us `n`, `e`, and `ct`. However, we find the real exploit in how the modulus was built. The hints point us away from padding or exponent tricks and toward factorization, and the challenge title suggests that the prime factors are "clustered" rather than spread out. Once we estimate the fourth root of `n`, it becomes clear that the modulus is the product of four very close primes. From there the exploit path is straightforward: we need to search for divisors in a narrow window around that fourth root, recover all four factors, compute the multi-prime totient, derive `d = e^-1 mod phi(n)`, and decrypt the ciphertext to recover the flag.

## Solve

We are given `message.txt`:

```text
n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
e = 65537
ct = 2630159242114455882250729812770100011736485763047361297871782218963814793905003742546116295910618429
```

Together with the hints:

1. RSA usually means two primes, but what if someone got greedy?
2. Prime factors decomposition.

that tells us the problem is the modulus, not the exponent, padding, or message format. The title also nudges us in the same direction: if the primes are "clustered," we should expect them to sit very close to a common size.

The first thing we want from our solver is a rough estimate of that size. A simple fourth-root helper is enough:

```python
n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651

def iroot4(x):
    lo, hi = 0, 1 << ((x.bit_length() + 3) // 4 + 1)
    while lo + 1 < hi:
        mid = (lo + hi) // 2
        if mid ** 4 <= x:
            lo = mid
        else:
            hi = mid
    return lo

approx = iroot4(n)
print(approx)
```

When we run that, we land at:

```text
9671406556917033398285235
```

Instead of throwing generic factorization tools at `n`, we can try searching for prime divisors in a narrow window around that fourth root:

```python
from sympy import isprime

factors = []
remaining = n

for cand in range(approx - 500000, approx + 500000):
    if remaining % cand == 0 and isprime(cand):
        factors.append(cand)
        remaining //= cand

if isprime(remaining):
    factors.append(remaining)

print(factors)
```

The first factor we recover that way is:

```text
9671406556917033397931773
```

Once we divide by that factor and repeat the same idea on the quotient, we get the rest:

```text
9671406556917033398314601
9671406556917033398439721
9671406556917033398454847
```

So the modulus was:

```text
n =
9671406556917033397931773 *
9671406556917033398314601 *
9671406556917033398439721 *
9671406556917033398454847
```

At that point the hard part is over, from there we can just do ordinary RSA decryption with the multi-prime totient:

```python
from sympy import mod_inverse

e = 65537
ct = 2630159242114455882250729812770100011736485763047361297871782218963814793905003742546116295910618429
factors = [
    9671406556917033397931773,
    9671406556917033398314601,
    9671406556917033398439721,
    9671406556917033398454847,
]

phi = 1
for p in factors:
    phi *= (p - 1)

d = mod_inverse(e, phi)
m = pow(ct, d, n)
print(bytes.fromhex(format(m, "x")).decode())
```

Running that gives us:

```text
picoCTF{mul71_rsa_xxxxxxxx}
```

RSA itself is fine here. What is broken is the way this specific modulus was generated. By using four primes that are all nearly the same size, we turn the challenge into a guided local search once we look near the fourth root.