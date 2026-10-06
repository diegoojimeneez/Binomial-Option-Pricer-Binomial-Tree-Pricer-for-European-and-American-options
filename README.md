# Binomial Option Pricer

A Cox-Ross-Rubinstein binomial tree pricer for European and American
options, written in NumPy. Includes Greeks by finite difference and a
Black-Scholes reference implementation for validation.

## Features

- European and American exercise
- Calls and puts
- CRR parameterization: `u = exp(σ√Δt)`, `d = 1/u`
- Greeks: delta, gamma, vega, theta
- Black-Scholes closed form for comparison
- No-arbitrage guard on the risk-neutral probability

## Usage

```python
option = Option_Pricer(s0=100, K=100, T=1, r=0.05, sigma=0.2,
                       n=500, opt="c", style="european")
option.opt_price(500)
option.delta()
```

## Method

Terminal prices were calculated with s0 * (u ^ j) * (d ^ (n-j)) to get the payoff of an option for a given strike price and option type. Backward induction was used to get the value of the option starting at the last step (n) finishing at step 0 by multiplying the discount factor by the risk-neutral probability of an up move and a down move on the option's value: e^{-rΔt}[p*V_up + (1-p)*V_down]. Risk-neutral probabilities make the stock's expected growth equal the risk-free rate. The option's value equals the cost of a portfolio holding the underlying stock and cash that reproduces its payoff.

## Validation

| Check | Result |
|---|---|
| Hand-built 3-step tree | 7.475, matches at every node |
| Hull worked example | 1.2823 call / 1.0592 put |
| Put-call parity | Holds to machine precision at all `n` |
| Convergence to Black-Scholes | 10.4466 at `n=500` vs 10.4506 analytic |
| American call = European call | Identical, as theory requires |
| American put > European put | Gap 0.5193 at `r=5%`, widening with `r` |
| Delta vs analytic | 0.63677 vs 0.63683 |
| Vega vs analytic | 37.5052 vs 37.5240 |
| Theta vs analytic | −0.017567 vs −0.017573 per day |

![Convergence](convergence.png)

An option's payoff has a sharp corner at the strike price. When the steps are even there's a node in the tree equal to K, when n is odd the strike price falls between two nodes. Therefore, the price alternates between being a notch too high or too low depending on the step.

![Greeks](greeks.png)

Delta: delta is computed by taking the slope of the option value. We can see how it assimilates to the cumulative distribution function, where with low stock prices the option delta was moving almost at par with the stock. 
Gamma: gamma was computed by taking the slope of the delta function. As the stock approaches strike price gamma shoots up and peaks. 
Vega: vega is the change in price per unit of volatility. At the strike price, a change in volatility could put me far in the money or far out of the money. That is why we see such a high vega when the stock approaches strike price. Volatility matters more at the strike price than anywhere else.
Theta: theta measures time decay, or how the options value changes as maturity is approached. When far out of the money, the options value does not change much as the value is already very low. Far in the money, with r > 0, the option's value increases as maturity is closer and the put is very likely to be exercised. The strike then acts like something that is owed to whoever owns the put, and the discount shrinks as expiry nears, increasing the value of the option as maturity nears. If r=0 the discount disappears. However, when it's slighlty out of the money, it decreases the most, as uncertainty of what is going to happen is at its highest.

The tree requires `d < exp(rΔt) < u`, which under CRR reduces to

    r·√Δt < σ

If violated, the risk-neutral probability falls outside [0, 1] and the
model returns meaningless prices without error. Example: `r=0.5,
σ=0.02, n=1` gives `p = 16.71` and a price of −18.87.

A guard raises `ValueError` when `p` leaves [0, 1].

Note this depends on `n`: the same `r` and `σ` can be invalid at a
coarse tree and valid at a fine one, since `√Δt` shrinks as steps are
added.

## Known issue: gamma

Gamma by central difference on `s0` is unstable, because a binomial
tree's price is not smooth in `s0` — the lattice shifts rather than
sliding along a curve.

| h | Gamma |
|---|---|
| 0.01 | 3.354568 |
| 0.1 | 0.335457 |
| 0.5 | 0.067091 |
| 1 | 0.033546 |
| 2 | 0.020309 |
| Black-Scholes | 0.018762 |

The value does not converge as `h` shrinks; it diverges. The proper fix
is to read gamma off the tree's own nodes at step 2 rather than
repricing. Not implemented.

## Not implemented

- Dividends
- Implied volatility
- Tree-node Greeks

## References

Hull, *Options, Futures and Other Derivatives*, ch. on binomial trees.
Cox, Ross and Rubinstein (1979).
