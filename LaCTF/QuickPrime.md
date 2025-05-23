# Challenge

In this challenge, two primes p and q were generated as successive prime terms generated by a linear congruential generator with equation ```x_(n+1) = ax_n + c mod m```. The goal was to factor an RSA key N = pq with this information.

Here is the source code of the challenge:

```python

#!/usr/local/bin/python3

from secrets import randbits
from Crypto.Util.number import isPrime


class LCG:

    def __init__(self, a: int, c: int, m: int, seed: int):
        self.a = a
        self.c = c
        self.m = m
        self.state = seed

    def next(self):
        self.state = (self.a * self.state + self.c) % self.m
        return self.state


while True:
    a = randbits(512)
    c = randbits(512)
    m = 1 << 512
    seed = randbits(512)
    initial_iters = randbits(16)
    # https://en.wikipedia.org/wiki/Linear_congruential_generator#m_a_power_of_2,_c_%E2%89%A0_0
    if (c != 0 and c % 2 == 1) and (a % 4 == 1):
        print(f"LCG coefficients:\na={a}\nc={c}\nm={m}")
        break

L = LCG(a, c, m, seed)
for i in range(initial_iters):
    L.next()

P = []
while len(P) < 2:
    test = L.next()
    if isPrime(test):
        P.append(test)

p, q = P

n = p * q

t = (p - 1) * (q - 1)

e = 65537

d = pow(e, -1, t)

message = int.from_bytes(open("flag.txt", "rb").read(), "big")

ct = pow(message, e, n)

print(f"n={n}")
print(f"ct={ct}")
```

# Solution

Without loss of generality, by symmetry between p and q, we can suppose that q comes after p in the generated pseudo-random sequence. Then, q can be written as ```C_1(k) p + C_2(k) mod m``` where C_1 and C_2 are two constants depending on the number of iterations k that were made in the LCG to reach q from p.
Thus, we can write ```N = C_1(k)p^2 + C_2(k)p mod m``` which is a quadratic equation with unknown p mod m for fixed k.
Since m is a power of 2, this can be solved efficiently with Hensel lifting.
So the walkthrough is the following:
- try for different values of k
- compute ```C_1(k)``` and ```C_2(k)```
- compute a solution of the quadratic equation mod m
- check if the gcd of this solution with N is strictly greater than 1 and strictly smaller than N

Here is the code associated with the solution:

```python

from math import gcd

a=11133339924092871626712363559494474676108690932794398616385672726064443904046989773127456832304449143798514950158407176476431913207010148674959914687055589
c=11648424987579142712677810266560105932217003752521567814504247084767215516445631558002499709171808204172225792977915876750882111983439221746279456484560169
m=13407807929942597099574024998205846127479365820592393377723561443721764030073546976801874298166903427690031858186486050853753882811946569946433649006084096
n=10875734392264302823715767173486212139748091012485826395016151925665191030437566678737909602230473363352969575880920190321548997905412685294976530401191301975183101362837833971227415187172099427190835510283422120341997693180841837586270191596468323384467210478267558677363882356938351355257897871142939938289
ct=9833715149661437967226749181186569522636399862563324417472742139133446562816277072862539307805499681138905954837091209992763355942017210360892725002273773869751215165646393222544849930914762702420095658198512342934712644494272008495190307903718226238105007769497626782133720043283826867042816485160058682301


from Crypto.Util.number import inverse, long_to_bytes
import time

def modular_sqrt(a, p):
    """ Finds square root of a mod p when p is a prime power of 2 using Hensel lifting """
    if p == 2:
        return a % 2  # sqrt(a) mod 2 is simply a % 2

    # Start with mod 2
    x = a % 2
    q = 2

    while q < p:
        q *= 2
        if (x * x - a) % q != 0:
            x += q // 2  # Adjust with Hensel's Lemma
    
    return x


def lift_sol(a, b, c, sol_lists, n):
    # Lifts the solutions from mod 2^n to mod 2^(n+1)
    # sol_lists is the list of solutions mod 2^n
    res = []
    for x0 in sol_lists:
        z = (a*x0**2 + b*x0 + c) % pow(2, n+1)
        z = z // pow(2,n)
        if z%2 == 0 and b%2 == 0:
            res.append(x0)
            res.append((x0+pow(2,n))%pow(2,n+1))
        elif b%2 == 0:
            # pas de solution
            pass
        else: # b est impair
            res.append((x0+pow(2,n)*z)%pow(2,n+1))
    return res

def solve_quadratic_modulo_2n(a, b, c, n=512):
    # Solves ax^2 + bx + c = 0 mod 2^n
    res = []
    
    # Compute the first solution mod 2
    if c%2 == 1:
        if (a+b)%2 == 1:
            res.append(1)
    elif c%2 == 0:
        res.append(0)
    # Lifting

    for n in range(1,512):
        res = lift_sol(a, b, c, res, n)

    return res


def build_params_list(a,n,c,K,m):
    # Build the params list up to the rank k
    params_list = []
    constant_coeff = (-n)%m
    deg_1_coeff = 0
    for j in range(K):
        deg_1_coeff += ((c*pow(a,j,m))%m)
        deg_1_coeff = deg_1_coeff % m
        deg_2_coeff = pow(a,j+1,m)
        params_list.append((deg_2_coeff, deg_1_coeff, constant_coeff))
    return params_list

start = time.time()

K = 15000
params_list = build_params_list(a,n,c,K,m)

for params in params_list:
    deg_2, deg_1, deg_0 = params
    solutions = solve_quadratic_modulo_2n(deg_2, deg_1, deg_0)
    if len(solutions) >= 1:
        for s in solutions:
            if gcd(s,n) > 1:
                print(s)

# We find the following factor
factor = 990049761718942512224666502446631814318235696574620230653600376427064573496756598018201683694725663641180399938661723118741231661213653423741391216602549

q = n//factor
phi = (factor-1)*(q-1)
d = pow(65537,-1,phi)

from Crypto.Util.number import long_to_bytes

print(long_to_bytes(pow(ct, d, n)))
```
