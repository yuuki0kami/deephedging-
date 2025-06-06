# Deep Hedging - Agent Architecture
## Description of the default Deep Hedging Agent 
#Deep Hedging フレームワークにおける 「エージェント（ネットワーク）」の構造と定義を詳細に記述した設計書

This document describes the default agent used in the Deep Hedging code base. This is _not_ the agent used in our [Deep Hedging paper](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3120710) but is a more advanced version. 

The Deep Hedging problem for a horizon $T$ hedged over $M$ time steps with $N$ hedging instruments is finding an optimal *action* function $a$ as a function of feature states $s_0,\ldots,s_{T-1}$ which solves

$$
 \sup_a:\ \mathrm{U}\left[\ 
    Z_T + \sum_{t=0}^{T-1} a(s_t) \cdot DH_t + \gamma_t \cdot | a(s_t) H_t |
 \ \right] \ .
$$

c.f. the description in the [main document](README.md). The objective function $\mathrm{U}$ is a _OCE monetary utility_ which is itself given as 

$$
    \mathrm{U}(X) := \sup_y:\ \mathrm{E}\left[
        u(X+y) - y
    \right]
$$

A number of different functions $u$ are supported in `objectives.py`. Classic choices are the entropy with $u(x) = (1-\exp(-\lambda x))/\lambda$ or CVaR with $u(x)=(1+\lambda) \min(0, x)$. This corresponds to CVaR@p at confidence $p = \lambda/(1+\lambda)$. For example $\lambda=1$ corresponds to CVaR@50%.

The **agent** in our problem setup is the functional $a$ which maps the current **state** $s_t$ to an **action**, which in this case is simply how many units of the $n$ hedging instruments to buy in $t$. In practise, $s_t$ represents the features selected from the available features at time $t$.  

### Initial Delta

The action at time zero is actually given by $a_0$ from the network described above plus an initial delta $a^\mathrm{init}$. The reason for this is that unless the portfolio $Z_T$ provided already contains the initial hedge, this hedge will look very different then subsequent hedges. Therefore, it is easier to train an additional $a^\mathrm{init}$. 

### Recurrence


The agent provided in ``agent.py`` provides for both "recurrent" and non-recurrent features. It should be noted that since the state at time $s_t$ contains the previous action $a_{t-1}$ as well as the aggregate position $\delta_{t-1}$ strictly speaking even a "non-recurrent" agent is actually recurrent.

Define the function

$$
    \mathbf{1}(x) := \left\\{ \begin{array}{ll} 1 & \mathrm{if}\ x > 0.5\\
                                    0 & \mathrm{else.} 
                                    \end{array}\right.
$$

with values in $\\{0,1\\}$.

### Recurrent State Types

**Classic Recurrent States** are given by

$$
   h_t = \mathrm{tanh} F(s_t, h_{t-1}) 
$$

where $F$ is a neural network. This is the original recurrent network formulation and suffers from both gradient explosion and long term memory loss. To alleviate this somewhat we restrict $h_t$ to $(-1,+1)$.

**Aggregate States** represent aggregate statistics of the path, such as realized vol, skew or another functions of the path. The prototype exponential aggregation function for such a hidden state $h$ is given as 

$$
   h_t = h_{t-1} (1 - z_t ) + z_t F(s_t, h_{t-1})  \ \ \ \ z_t \in [0,1]
$$ 

where $F$ is a neural network, and where $z_t$ is an "update gate vector". This is also known as a "gated recurrent unit" and is similar to an LSTM node. 

In quant finance such states are often written in diffusion notation with $z_t \equiv \kappa_t dt$ where $\kappa_t\geq 0$ is a mean-reversion speed. In this case the $dt\downarrow 0$ limit becomes

$$
    h_t = e^{-\int_0^t \kappa_sds} h_0 + \int_0^t \kappa_u e^{-\int_u^t \kappa_sds} F(s_u,h_{u-})du \ .
$$

The appendix provides the derivation of this formula from its associated SDE.

**Past Representation States:** information gathered at a fixed points in time, for example the spot level at a reset date. Such data are not accessible before their observation time.
The prototype equation for such a process is

$$
 h_t = h_{t-1} (1 - z_t ) + z_t F(s_t, h_{t-1}) \ \ \ \ z_t \in \\{0,1\\}
$$ 
 
which looks similar as before but where $z_t$ now only takes values in $\\{0,1\\}$. This allows encoding, for example, the spot level at a given fixed time $\tau$.

**Event States** which track e.g. whether a barrier was breached. This looks similar to the previous example, but the event itself has values in $\\{0,1\\}$.

$$
 h_t = h_{t-1} (1 - z_t ) + z_t \mathbf{1} \Big(  F(s_t, h_{t-1}) \Big)  \ \ \ \ z_t \in \\{0,1\\}
$$

where we need to make sure that $h_{-1}\in\\{0,1\\}$, too. 

## Definition of the Network

When using the code, the agent is defined as follows: let $s_t\in\mathbb{R}^{m_\mathrm{s}}$ feature state of the network, and assume we are modelling $m_\mathrm{c}$ classic states, $m_\mathrm{a}$ aggregate states, amd $m_\mathrm{r}$ past representation states. In case the sample path generated by the market are all identical at time step zero, then
the initial states are plain variables $h^c_{-1}\in\mathbb{R}^{m_c},\ldots,h^e_{-1}\in\mathbb{R}^{m_e}$ we will lean. See below comments for further information on how to handle non-trivial initial market states.

Let $m:=m_s+m_c+m_a+m_r+m_e$ be the dimension of the input vector for the network in each step.

At step $t$, assume now that $(s_t,h^c_{t-1},h^a_{t-1},h^r_{t-1},h^e_{t-1})\in \mathbb{R}^m$.
We call $w\in\mathbb{N}$ the `width` of the network, and $d\in\mathbb{N}$ its `depth`. Also assume that $\alpha:\mathbb{R}\rightarrow \mathbb{R}$ is an `activation` function with the usual convention of elementwise application for vector arguments. We will also use a `final_activation` function $\ell:\mathbb{R}\rightarrow\mathbb{R}$ which will usually be `linear`.

$$
    y^0 := \alpha\left( b^0 + W^0 \cdot \left( \begin{array}{c}
                s_t \\
                \mathrm{tanh}( h^c_{t-1} ) \\
                h^a_{t-1} \\
                h^r_{t-1} \\
                \mathbf{1}( h^e_{t-1} ) \\
            \end{array}
    \right) \right) \ \ \ \ \mathrm{where\ we\ train}\ W^0 \in \mathbb{R}^{w,m}, b^0\in \mathbb{R}^w.
$$

and where $\alpha$ is an activation function. We then iteratively apply for $k=1,\ldots,d-1$:

$$
    y^{k+1} = \alpha\left( b^k + W^k \cdot y^k
    \right) \ \ \ \ \mathrm{where\ we\ train}\ W^k \in \mathbb{R}^{w,w}, b^k\in \mathbb{R}^w.
$$

Let now $M:=m_s+m_c+2m_a+2m_c+2m_e$.
In the final step we set

$$
    \left(
            \begin{array}{c}
                a_t \\
                F^c_t \\
                F^a_t \\
                F^r_t \\
                F^e_t \\
                z^a_t \\
                z^r_t \\
                z^e_t 
            \end{array}  
    \right)= \ell\left( b^{k+1} + W^{k+1} \cdot y^{k+1}
    \right) \ \ \ \ \mathrm{where\ we\ train}\ W^{k+1} \in \mathbb{R}^{M,w}, b^{k+1}\in \mathbb{R}^M.
$$

Here, $a_t$ is the action at time $t$. We then also define the hidden states at $t$ in line with above definition as

$$
    h^c_t := \mathrm{tanh}( F^c_t ) \ ,
$$

$$
    z^a_t := \mathrm{sigmoid}( \hat z^a_t ) \ \ \ \mathrm{and\ then} \ \ \ 
    h^a_t := h^a_{t-1} (1 - z^a_t) + z^a_t F^a_t\ ,
$$

$$
    z^r_t := \mathbf{1}( \hat z^r_t ) \ \ \ \mathrm{and\ then} \ \ \ 
    h^r_t := h^r_{t-1} (1 - z^r_t) + z^r_t F^r_t\ , 
$$

$$
    z^e_t := \mathbf{1}( \hat z^r_t ) \ \ \ \mathrm{and\ then} \ \ \ 
    h^e_t := h^e_{t-1} (1 - z^e_t) + z^e_t \mathbf{1}( F^e_t ) \ .
$$

## Configuring the Agent

The `agent.py` construction according to above is driven by the `agent` section of the `gym` `config`. The core network parameters are given by:

    config.gym.agent.network.activation = softplus
    config.gym.agent.network.final_activation
    config.gym.agent.network.depth = 5
    config.gym.agent.network.width = 20 

Define features used

    config.gym.agent.features = ['price', 'delta', 'time_left']

To activate recurrence, set any of the numbers $m_c+m_a+m_r+m_e$ to non-zero:

    config.gym.agent.recurrence.states.classic = m_c
    config.gym.agent.recurrence.states.aggregate = m_a
    config.gym.agent.recurrence.states.past_repr = m_r
    config.gym.agent.recurrence.states.event = m_e

The following flag can be used to enforce that $h_t^a \in (-1,+1)$:

    config.gym.agent.recurrence.bound_aggr_states = True

## Handling non-constant initial market states

Usually, a state $s_t$ has different values for different samples. However, the initial state $s_0$ often does not: if we start a market simulation for today, then "today" is fixed for each simulated path.

However, there are a number of applications where this is not necessarily the case:
1) Training with uncertainty in the initial step. An extreme case if we want to train a model to learn hedging a portfolio $Z_T$ for different current states. 
1) Training for a number of payoffs $Z_T$ at the same time. In this case, the portfolio $Z_T$ may change per sample.
2) Trainig for different risk-aversion parameters at the same time, to obtain an "efficient frontier".

The examples (2) and (3) are covered in https://arxiv.org/abs/2207.07467. Case (1) is supported by our  `world` simulator: it allows to start "today" in an (approximate) invariant state. In this case spot levels for the hedging instruments different at time $0$ across samples.

The variables affected in the Deep Hedging Framework here are:
* The variable $y$ in the OCE objective.
* The initial delta $a^\mathrm{init}$.
* The initial hidden states $h_{t-1}$ of our agents.

By default, they are all real variables, which corresponds to the case of a constant initial state $s_0$. The can all be turned into networks if this changes. The definitions follow the same logic as the network definitions above.

**OCE Intercept** $y$:

    config.gym.objective.y.network.activation. = relu 
    config.gym.objective.y.network.final_activation = linear
    config.gym.objective.y.network.depth = 3
    config.gym.objective.y.network.width = 10
    config.gym.objective.y.features = []

**Initial Delta** $a^\mathrm{init}$:

    config.gym.agent.init_delta.network.activation. = relu 
    config.gym.agent.init_delta.network.final_activation = linear
    config.gym.agent.init_delta.network.depth = 3
    config.gym.agent.init_delta.network.width = 10
    config.gym.agent.init_delta.features = []

To turn off using $a^\mathrm{init}$ use

    config.gym.agent.init_delta.active = False

**Initial State** $h_{-1}$:

    config.gym.agent.state.network.activation = relu
    config.gym.agent.state.network.final_activation = linear
    config.gym.agent.state.network.depth = 3
    config.gym.agent.state.network.width = 10
    config.gym.agent.state.features = []


## Appendix: Mean Reversion Math

### Constant mean reversion
Consider

$$
    dx_t = \kappa (y_t - x_t)dt
    \ \ \ \ \Leftrightarrow \ \ \ \
    x_{t+dt} = \kappa dt\ y_t + (1 - \kappa dt )\ x_t
$$

Then

$$
    d\left( e^{\kappa t} x_t \right)
        = \kappa e^{\kappa t} x_tdt +
        \kappa e^{\kappa t}(y_t - x_t)dt = \kappa e^{\kappa t}y_tdt
$$

and therefore

$$
    x_t = e^{-\kappa t} x_0 + \kappa \int_0^t e^{-\kappa(t-s)} y_sds
$$

### Mean reversion as function of time

Let

$$
dx_t = \kappa_t (y_t - x_t)dt
$$

Set $K_t:=\exp(\int_0^t \kappa_sds)$. Then

$$
    d\left( K_t x_t \right)
        = \kappa_t K_t x_tdt +
        \kappa_t K_t (y_t - x_t)dt = \kappa_t K_t y_tdt
$$

and therefore

$$
    x_t = e^{-\int_0^t k_sds} x_0
        + \int_0^t \kappa_re^{-\int_r^t\kappa_sds } y_sds
$$
