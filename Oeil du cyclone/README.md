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
TARGET_HASH_PREFIX = "f687cb74fdcefefc"

# (Station data from the challenge...)
x_intact = [1, 2, 3, 4, 5]
y_intact = [0xd0393f...f5, 0xa0deb8...e4, 0x9dc6d6...45, 0xff3a80...57, 0x748755...cb]
x_flooded = [6, 7, 8, 9]
y_partial = [0x18e91e...00, 0x2a67e4...00, 0xdbd27a...00, 0x57e86b...00]

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
K6, K7, K8, K9 = [(lagrange_eval(x, x_intact, y_intact, p) - y_partial[i]) % p for i, x in enumerate(x_flooded)]

def to_signed(val): return val - p if val > p // 2 else val
V7, V8, V9 = map(to_signed, [(K7 - 6*K6) % p, (K8 - 21*K6) % p, (K9 - 56*K6) % p])

for e6 in range(2**20):
    e7, e8, e9 = 6*e6 + V7, 21*e6 + V8, 56*e6 + V9
    if 0 <= e7 < 2**20 and 0 <= e8 < 2**20 and 0 <= e9 < 2**20:
        c = (e6 - K6) * pow(120, -1, p) % p
        secret = (A0 - 120 * c) % p
        flag_candidate = long_to_bytes(secret)
        if hashlib.sha256(flag_candidate).hexdigest().startswith(TARGET_HASH_PREFIX):
            print(f"Flag found: {flag_candidate.decode()}")
            break
```

> CCOI26{CycL0n3_B3l4l_R3uN10n_974}
