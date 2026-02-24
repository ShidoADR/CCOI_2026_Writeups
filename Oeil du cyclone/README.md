# __CCOI26__ 
## _Oeil du cyclone (Eye of the Storm)_

## Information
**Category:** | **Points:** | **Writeup Author**
--- | --- | ---
Crypto | 200 | Moshimoshi

**Description:** 

> Cyclone Garance has devastated Reunion Island. The activation code for the emergency evacuation system was protected by a secret sharing scheme distributed among 9 weather stations. The reconstruction threshold is 6 fragments. 
> 
> 5 high-altitude stations are intact, but 4 coastal stations were flooded: their data lost the least significant bits (lowest 20 bits).
>
> **Flag format:** CCOI26{...}

## Solution

### 1. Mathematical Analysis
The challenge uses Shamir’s Secret Sharing (SSS) over a Mersenne Prime $p = 2^{521} - 1$. We have a threshold $k=6$, meaning the secret is the constant term $a_0$ of a degree 5 polynomial:
$P(x) = a_5x^5 + a_4x^4 + a_3x^3 + a_2x^2 + a_1x + a_0 \pmod p$

I have 5 intact points $(x_1, y_1) \dots (x_5, y_5)$. Since I need 6 points to solve the system, I have one degree of freedom. I can represent the polynomial using Lagrange interpolation of the 5 known points plus an unknown leading coefficient $c$:
$P(x) = L(x) + c \cdot Q(x) \pmod p$
Where $Q(x) = (x-1)(x-2)(x-3)(x-4)(x-5)$.

### 2. Handling the "Flooded" Data
For the 4 flooded stations (x = 6, 7, 8, 9), I only have partial values $\tilde{y}_i$. The true values are $y_i = \tilde{y}_i + e_i$, where the error $e_i$ is small ($0 \leq e_i < 2^{20}$).

Substituting $x=6$ into the equation:
$e_6 \equiv (L(6) - \tilde{y}_6) + c \cdot Q(6) \pmod p$
Let $K_i = (L(x_i) - \tilde{y}_i) \pmod p$. For the flooded stations, I get:
- $e_6 \equiv K_6 + 120c \pmod p$
- $e_7 \equiv K_7 + 720c \pmod p$
- $e_8 \equiv K_8 + 2520c \pmod p$
- $e_9 \equiv K_9 + 6720c \pmod p$

### 3. Attack Strategy: Brute Force
Since $c$ is constant, I can eliminate it. For example: $e_7 - 6e_6 \equiv K_7 - 6K_6 \pmod p$.
Because the errors $e_i$ are very small compared to the huge prime $p$, these modular congruences behave like linear equations over integers.

I decided to brute force the value of $e_6$ (from $0$ to $2^{20}$). For each $e_6$, I calculate the implied $e_7, e_8,$ and $e_9$. If all of them fall within the valid range $[0, 2^{20}-1]$, I've found the correct coefficient $c$ and, consequently, the secret.

### 4. Implementation & Flag
I wrote a Python script to perform the interpolation, iterate through the 20-bit error range, and verify the result against the provided SHA256 prefix.

```python
from Crypto.Util.number import long_to_bytes
import hashlib

p = 2**521 - 1
TARGET_HASH = "f687cb74fdcefefc" 

x_intact = [1, 2, 3, 4, 5]
y_intact = [
    0xd0393fd5aa76c02f53757a5883d97a0f0ade112cffc590c8378f2b5a6696a284dcc1ef10c29f7275958952bca3c40922f75258f47e808d587aca867f48f0d798f5,
    0xa0deb8650c459c78e99ca5ae29c1399c8221723e6c966a4a4494ec69bcb20399336bba13c10998b4b0b554cffdaec9b8b536e6fa9ea4eefa7321782797b84672e4,
    0x9dc6d639cbda2c6893efafe086027e1f9126a9e27f2d342e45e8090675c2eca7e4ae330b163f8f059fa665a20ea4be41a4de9fe882ac3b08387ba8649622293745,
    0xff3a80c762b7a71ee3793ed87a7951f819960a86b067cefbe94cac78b9f556291ebf42ae21395da1a5e9d3d426624b6cf5bebb4487d9311737417749e401c0cb57,
    0x748755843bdf0733e28882bb8f096fdd4c4ae2142cba5fb2ea4ba7e65a7b007a75f34a4f7a94b4b8e5b9d425d415b5750066cb52e451f11933b086614b816d4ecb
]

x_flooded = [6, 7, 8, 9]
y_partial = [
    0x18e91e304d2372e99ce65481f4a15284c423aa9ac47a25b639109b2c0c5d60cb6ba133679b80d2d34cfdc2c2968c5b83977eaa1b6e5ad7ed0368e3d0a9639300000,
    0x2a67e416cef50a7fd1040a3c88f446f6955c3564ef1992c7311eab32fc23958dcbb2918c2ff4897a9380dcf879b81f599b4c34142f81454279da4cdb6245300000,
    0xdbd27adc2803b734baba0522d86af830f98ee4051f093dd8a86cd68f8366481c71859657bcaaf62d8e20cde862d85e4e66e580aff9ee9a2e558135fef75c500000,
    0x57e86be63e6ca409bbf147ebcd20ae61d581cec154bd076cddf821be5bd0fcc42db742bb80174af1bb5c773ec91e2884c5d273125030417e2c0ecb961be6800000
]

def lagrange_eval(x_eval, xs, ys, p):
    res = 0
    for i in range(len(xs)):
        num, den = 1, 1
        for j in range(len(xs)):
            if i != j:
                num = (num * (x_eval - xs[j])) % p
                den = (den * (xs[i] - xs[j])) % p
        term = (ys[i] * num * pow(den, -1, p)) % p
        res = (res + term) % p
    return res

A0 = lagrange_eval(0, x_intact, y_intact, p)
Ks = [(lagrange_eval(x, x_intact, y_intact, p) - y_partial[i]) % p for i, x in enumerate(x_flooded)]

V7 = (Ks[1] - 6 * Ks[0]) % p
V8 = (Ks[2] - 21 * Ks[0]) % p
V9 = (Ks[3] - 56 * Ks[0]) % p
Vs = [v - p if v > p // 2 else v for v in [V7, V8, V9]]

print("[*] Recherche du flag en cours...")
for e6 in range(2**20):
    e7, e8, e9 = 6*e6 + Vs[0], 21*e6 + Vs[1], 56*e6 + Vs[2]
    
    if 0 <= e7 < 2**20 and 0 <= e8 < 2**20 and 0 <= e9 < 2**20:
        c = (e6 - Ks[0]) * pow(120, -1, p) % p
        secret = (A0 - 120 * c) % p
        flag = long_to_bytes(secret)
        
        if hashlib.sha256(flag).hexdigest().startswith(TARGET_HASH):
            print(f"\n[+] Succès !")
            print(f"Le flag est : {flag.decode()}")
            break
```

> CCOI26{CycL0n3_B3l4l_R3uN10n_974}
