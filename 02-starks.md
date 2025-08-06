# STARKs, DEEP and FRI
This document describes some essential aspects of zk-STARK constructions, with a strong focus on DEEP and FRI. Fiat-Shamir, blinding and some details are not covered in as much depth. This document is based on my understanding and there may be mistakes in it. 

>[!WARNING]
> There are mistakes in this article.
> It's being actively worked on and this warning will
> be removed once the description of STARKs is sound.

We will be working with just a single column / variable and therefore just a single trace polynomial `T(x)` and just one transition constraint `C(x)` (no boundary constraints).

## Trace Polynomials
A computation trace is the state of each variable used in that computation at each step. We can represent a trace as a table with rows and columns, where each row represents a point in time and each column represents a variable. In STARK constructions, we interpolate each column / each variable as a polynomial, where the domain is a set of `n` roots of unity.

Example trace:

| index |  x  |
|-------|-----|
| w_0   |  2  |
| w_1   |  4  |
| w_2   |  16 |
| ...   |     |
| w_n   | ... |

Where `w_n` are roots of unity for the domain size and finite field. 
When using `arkworks`, these can be obtained from:

```rust
let domain = GeneralEvaluationDomain::<Fr>::new(trace_len).unwrap();
```

Later we will want to evaluate over a shifted, extended domain `S` that doesn't intersect our original domain `H`.

```rust
let extended_domain = GeneralEvaluationDomain::<Fr>::new(trace_len * 8).unwrap();
```
gives us an extended domain that overlaps with our original domain. The shifted extended domain would have no overlap with our original domain, which prevents us from accidentally leaking trace values or dividing by zero when sampling:

```rust
let shifted_domain = extended_domain.get_coset(Fr::from(7)).unwrap();
```



## Constraint Polynomials
Constraint functions are written in such a way that they represent polynomials.

`ci(x) = T(gx) - T(x)^2`

encodes a constraint where the next row for a column must be the square of the previous row, as seen in the trace table above.

the constraints are satisfied for this colum that represents the computation trace of the x variable.

We construct a composite constraint polynomial C(x) that must be interpolated over the extended domain, satisfying constraints over the original domain for a valid trace.
Note that in our examples we don't need a composite constraint polynomial, since we only work with a single transition constraint C(x).

`C(x) = ∑ c1(x) + c2(x) + ... + ci(x)`

Note that in production systems we add random coefficients for extra security.

## LDE - Low degree extension
We use low degree extension (LDE) to evaluate our constraints at `T(x), x∈H'`, where H' is the extended domain that intersects our original domain / roots of unity H.

Remember that our constraints evaluate to `0` over the original domain, because they are satisfied for the computation trace. This is not the case for evaluations `C(T(x), T(gx), ...)` over the extended domain `x∈H'`.

## Quotient
The quotient polynomial Q(x) is computed by dividing the composite constraint polynomial C(x) by the vanishing polynomial for the original domain (roots of unity). => `C(x) / Z(x) = Q(x)`. Note that this will only yield a low-degree Q(x) if the trace satisfies the constraints over the original domain, since C(x) was interpolated over the extended domain that includes the original domin.

Spot checks are however only performed in and outside of the extended domain `x∈H', x∌H, z∌S, z∌H'`. If we tried to spot-check inside the original domain `H`, then we would leak trace values which is not zero-knowledge.

## DEEP
In STARK constructions we want to prove that we know a trace that satisfies our constraints over the original domain without revealing any of the original trace values.

To achieve this, we have to construct a DEEP polynomial from the trace and constraints.

`D(x) = ((Q(x) - Q(z)) / x - z)`

Where `Z(x)` is the vanishing polynomial of the original domain and `Q(x)` is the quotient polynomial from `C(x) / Z(x)`. Note that in a real-world scenario we would add trace evaluations:


`D(x) = alpha_i * ((Q(x) - Q(z)) / x - z) + alpha_j * ((T(x) - T(z) / x - z)`

We use FRI to check, in a point-wise manner, that the polynomial `Q(x)`, of degree `d`, when divided by a degree-1 polynomial (x - z), yields a low-degree polynomial that is within our degree bound, which depends on the trace size and the degree of our constraints.

Next we commit the folding steps in the prover to a merkle tree and for each spot check / random challenge (x, z) the verifier must check the commitments against the merkle root that is part of the proof output.

> [!NOTE]
> If we tried to interpolate C(x) over the original domain as a polynomial, then we would get the zero polynomial for a valid trace. That makes it impossible to reason about the trace through LDE. 
> Remember not to interpolate C(x) from evaluations over just the original domain if all constraints are always satisfied!


## Spot Checks 
In addition to checking that the DEEP polynomial is low-degree, we must also perform spot checks in the extended domain. This is usually all based on Fiat-Shamir challenges, but the underlying logic of the checks remains the same:

```rust
if constraint_poly.evaluate(z) != quotient_poly.evaluate(z) * vanishing_poly.evaluate(z) {
    panic!("Spot check failed!");
}

if constraint_poly.evaluate(x)
    != fibonacci_constraint(
        trace_poly.evaluate(g * g * x),
        trace_poly.evaluate(g * x),
        trace_poly.evaluate(x),
    )
{
    panic!("Spot check failed!");
}

// x∈H'∌H, z∌H'∌S
```

Where `g` is the generator of the original domain and `fibonacci_constriant` is an example of a raw constraint function that was used earlier when evaluating C(x) over trace polynomial evaluations.

The spot checks ensure consistency with the constraint logic and correctness of the quotient polynomial. Note that in most STARK constructions neither of these polynomials need to be interpolated, I just chose to interpolate them for simplicity during my research.

## FRI
FRI folding is an essential part of STARK constructions. At each step, it halves the evaluation domain and roughly halves the polynomial’s degree.

We use FRI to verify that the degree of D(x) is <= d, where d is the degree bound for our constraints and trace size. If this check succeeds for all our random challenges (x, z) and if the prover can provide valid merkle proofs for the evaluations of T(x) and the results of the FRI folding steps for the extended domain (we re-use x for these spot checks, so ideally we choose `x∈H'`), 

then we know with high certainty that the prover knows a trace that makes the constraint polynomial vanish over the original domain. This is equivalent to knowing a valid solution to our computational problem / having executed the program.

## Final implementation plan
Assuming only one column (one trace polynomial T(x)) for the fibonacci example.

### Prover

1. Interpolate `T(x)` over the original domain using the computation trace.
2. Interpolate `C(x) = fibonacci(T(ggx), T(gx), T(x))` over the extended domain, note that `T(ggx), T(gx), T(x)` will be revealed by the prover who previously committed these LDE evaluations to a merkle tree.
3. Commit to `Q(x) = C(x) / Z(x)` over the shifted domain.
3. Fold `D(x)` over the shifted domain using FRI.
4. Commit `Q(z)` for a random point `z` outside the domains.

### Verifier
1. Check that `Q(x) = fibonacci(T(ggx), T(gx), T(x)) / Z(x)`, for a few spot checks `x∈H'∌H`.
2. Check that `Q(z) = C(z) / Z(z)` at a random spot check `z∌H'∌S`.
2. Check that `D(x) = alpha * (Q(x) - Q(z)) / (x - z)` for a few spot checks `x∈S`.
3. Check that `D(x)` is a low degree polynomial by spot-checking each folding layer at the same spot checks `x∈S`.

The verifier only opens the merkle tree for the respective evaluations at the query points. The prover does not disclose an interpolated Q(x) or C(x) - actually interpolation is not required at all, 
except when building T(x) over the original domain.

It is difficult for a prover to forge points Q(z), C(z) that satisfy C(z) / Z(z) = Q(z), where Q(z) remains consistent with the DEEP formula for query points Q(x), `x∈S`.
This DEEP consistency check is essential to the security of our STARK.


## Toyni STARK
I am working on a toy implementation of STARK constructions [here](https://github.com/jonas089/toyni). This article is closely aligned with the code that I am writing in that repository, so please open an issue if you discover an inaccuracy or mistake in my current understanding of DEEP / FRI. Toyni is an active work in progress and will undergo major changes to closely align with this article.

>[!WARNING]
> In production STARKs we use randomness to blind the trace. 
> For example, the values alpha and z are derived using fiat-shamir, after the prover already committed to the
> LDE of the Trace. This makes it difficult for the prover to forge a polynomial that satisfys the verifier checks
> at just z and a few points in the extended domain.
