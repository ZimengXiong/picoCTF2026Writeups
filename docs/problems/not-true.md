
# Not TRUe

`Not TRUe` is a cryptography challenge built around a deliberately weak NTRU-like encryption scheme, and the provided `encrypt.py` more or less tells us how to break it. The code shows a very small ring dimension `N = 48`, small moduli `p = 3` and `q = 509`, and private polynomials with ternary coefficients in `{-1,0,1}`. Those choices make the secret key short enough to recover with a standard NTRU lattice attack. The exploit path is to read the construction from the source, turn the public key relation `h = p * f^-1 * g mod q` into the usual NTRU lattice, reduce that lattice with LLL and BKZ, filter the short vectors until we find a valid invertible private polynomial `f`, and then use `f` to decrypt each ciphertext block and reconstruct the flag.

## Solve

We can see the weakness immediately in `encrypt.py`. The parameters are TINY:

```python
N = 48
p = 3
q = 509
```

The secret polynomials are also deliberately short:

```python
def gen_poly():
    return R([randint(-1,1) for _ in range(N)])
```

Then key generation gives us the exact public-key structure:

```python
def generate_keys():
    while True:
        f = gen_poly()
        g = gen_poly()
        try:
            f_p_inv = R_modp(f)**-1
            f_q_inv = R_modq(f)**-1
            break
        except:
            continue

    h = R_modq(p*(f_q_inv*g))
    private_key = (f, g, f_p_inv, f_q_inv)
    public_key = h
    return public_key, private_key
```

Encryption follows the standard NTRU pattern too:

```python
def encrypt(h, m):
    r = gen_poly()
    return R_modq(p*(h*r) + m)
```

From just those snippets, we already know the important facts:

- the ring is `Z[x] / (x^48 - 1)`
- the public key is `h = p * f^-1 * g mod q`
- the secrets are very short
- the dimension is small

It now becomes pratical to force a lattice attack. The generated `public.txt` shows us the instance we are solving:

```text
N = 48
p = 3
q = 509
h = [311, 273, 153, 392, 105, 426, 45, 159, 421, 30, 404, 38, 313, 480, 37, 195, 82, 382, 236, 230, 340, 66, 199, 121, 279, 273, 271, 22, 270, 227, 26, 253, 213, 408, 323, 263, 283, 168, 88, 164, 383, 247, 5, 313, 150, 170, 89, 235]
ct = [[231, 6, 348, 205, ...], ...]
```

Once we have `h`, we can build a solver that builds the standard row-basis lattice using the circulant matrix of the public key:

```python
def recover_private_key(h, n, p, q):
    H = [[h[(i - j) % n] for j in range(n)] for i in range(n)]
    basis = [[0] * (2 * n) for _ in range(2 * n)]
    for i in range(n):
        basis[i][i] = 1
    for i in range(n):
        for j in range(n):
            basis[i][n + j] = H[j][i]
    for i in range(n):
        basis[n + i][n + i] = q
```

Then we reduce that lattice:

```python
    reduced = IntegerMatrix.from_matrix(basis)
    LLL.reduction(reduced)
    BKZ.reduction(reduced, BKZ.Param(block_size=20))
```

The most important part is that we do not trust the first short row that falls out of reduction. We must explicitly test whether a candidate row really corresponds to a valid `(f, p*g)` pair, i.e.:

```python
    for row in range(2 * n):
        base = [reduced[row, col] for col in range(2 * n)]
        for sign in (1, -1):
            vec = [sign * x for x in base]
            f = vec[:n]
            pg = vec[n:]
            if max(abs(x) for x in f) > 2 or max(abs(x) for x in pg) > 20:
                continue
            if any(x % p for x in pg):
                continue
            g = [x // p for x in pg]
            if [x % q for x in conv(f, h, n)] != [(p * y) % q for y in g]:
                continue
            try:
                invert_mod_ring(f, n, p)
                invert_mod_ring(f, n, q)
            except Exception:
                continue
            return f, g
```

Those checks matter because a short lattice vector is not automatically the real private key. We only keep a candidate if it is 1/ small, 2/ divisible the right way, 3/ satisfies the public-key relation, and 4/ is invertible modulo both `p` and `q`.

Once we recover a valid `f`, decryption is straightforward:

```python
def decrypt(ct, f, n, p, q):
    f_p_inv = invert_mod_ring(f, n, p)
    bits = []
    for block in ct:
        a = center(conv(f, block, n, q), q)
        b = [x % p for x in a]
        m = conv(f_p_inv, b, n, p)
        bits.extend(m)
```

That turns the ciphertext blocks back into bits, then bytes, and finally the flag:

```text
picoCTF{th4ts_s0_N0t_TRU3_38a83032}
```

NTRU-style schemes depend heavily on parameter choices. Here, the dimension is too small and the secrets are too short, so the private key is visible to lattice reduction. We are not breaking lattice cryptography in general (that would be trouble), this only works bc the instance is deliberately undersized.