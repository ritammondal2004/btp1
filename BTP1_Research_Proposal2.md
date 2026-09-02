($v_t-v_{t-1}$)  

# BTP-1: Controlled Mathematical Model Comparison for Trading Strategy Optimization

## 1. Objective & Hypothesis
This thesis project establishes a rigorous, hypothesis-driven ablation study to evaluate the impact of temporal memory and risk-adjusted reward shaping on deep reinforcement learning (DRL) trading agents. The primary goal is to systematically isolate how temporal memory (LSTM) affects state representation, and how dense risk-adjusted rewards (Differential Sharpe Ratio) combined with turnover regularization mathematically stabilize policy updates under shifting market regimes.

---

## 2. The Research Gap
While DRL has been widely applied to algorithmic trading, there is a fundamental lack of systematic ablation studies that decouple the effects of architectural memory from reward design. A close examination of the literature reveals two intersecting gaps:

1. **The Realism and Market Friction Gap:** As noted by **Millea (2021, "Deep Reinforcement Learning for Trading—A Critical Survey", §12.1, p. 21)**, the literature suffers from a significant realism gap: *"Very few papers consider all three factors: transaction cost, slippage... and spread."* Agents trained in frictionless environments routinely learn to hyper-trade ("churn"), exploiting minor anomalies that are instantly wiped out by real-world transaction costs.
2. **The Risk-Blindness of Memory Models:** While LSTM-PPO has been proven highly effective at handling the Partial Observability (POMDP) of financial data compared to flat baselines **(Zou et al. 2023, "A Novel DRL Based Automated Stock Trading System...", §4.6, p. 9)**, it typically relies on naive profit-based reward signals ($v_t - v_{t-1}$). This forces the agent to maximize absolute returns while remaining entirely blind to drawdown risk.

**The Gap:** While deep reinforcement learning has seen significant application in quantitative finance, the literature lacks a rigorous, unified ablation study that isolates the effects of temporal memory, dense risk-adjusted rewards, and friction-regularized turnover penalties within a single algorithm. Many RL models simplify transaction costs as fixed, ignore the temporal credit assignment problem, or over-rely on absolute profit functions that fail during high volatility (Pippas et al., The Evolution of RL in QF, Section 2, Page 3). Furthermore, while LSTM-PPO has been shown to outperform standard PPO (Zou et al., A Novel DRL Based Automated Stock Trading System, Section 4.6, Page 9), it still largely relies on simple profit-based rewards rather than dense risk-adjusted signals. The Novelty: The precise novelty of this BTP1 is proving—through a mathematically controlled 4-step ablation study—that the Differential Sharpe Ratio (DSR) combined with a turnover penalty mathematically stabilizes LSTM-based PPO policy updates specifically during abnormal or highly volatile market drawdowns, effectively solving the "churning" problem in standard DRL trading.

**The Novelty:** This research bridges this gap by designing a controlled four-rung ladder. It introduces a **"Friction-Regularized DSR"** to explicitly combat the high-frequency turnover commonly induced by standard DRL agents when hunting for risk-adjusted returns. By doing so, it isolates the exact mathematical contribution of temporal memory, online risk-awareness, and friction-control within a single, defensible BTP-1 study.

> **📸 [SCREENSHOT PLACEHOLDER 1]** 
> * **Where to capture:** Millea 2021 ("Deep Reinforcement Learning for Trading—A Critical Survey"), Page 21, Section 12.1 "Common Ground".
> * **What to capture:** The specific bullet point that reads: *"Very few papers consider all three factors: transaction cost, slippage (when the market is moving so fast...), and spread..."*
> * **Where to paste:** Insert here to visually prove the market friction gap exists in current literature.

---

## 3. Walk-Forward Experimental Design & Ablation Structure

To ensure a mathematically controlled comparison—in line with strict "one variable at a time" hypothesis testing—the evaluation is broken down into two distinct phases. This prevents conflating the impact of state representation with reward design:

*   **Phase 1: Isolating Temporal Memory's Effect (Model 1 → Model 2)** 
    *   Holds the reward function constant (flat profit) and tests the specific impact of adding sequence-aware memory cells on the agent's ability to navigate non-stationary data.
*   **Phase 2: Isolating Reward Design's Effect (Model 2 → Model 3 → Model 4)** 
    *   Holds the architecture constant (LSTM) and iteratively tests the impact of risk-sensitivity (DSR), followed exclusively by the turnover friction penalty.

---

## 4. Detailed Mathematical Modelling

### 4.1. Base MDP Formulation
Across all four models, the underlying Markov Decision Process (MDP) is defined by the tuple $(\mathcal{S}, \mathcal{A}, \mathcal{P}, \mathcal{R}, \gamma)$:
*   **State ($s_t \in \mathcal{S}$):** A vector $s_t = [b_t, p_t, h_t, M_t, R_t, C_t, X_t]$ representing the cash balance, adjusted close price, shares owned, and technical indicators (MACD, RSI, CCI, ADX). 
*   **Action ($a_t \in \mathcal{A}$):** A continuous vector $a_t \in [-1, 1]^N$ denoting the target portfolio weights or trade proportions for $N$ assets.
*   **Portfolio Value ($v_t$):** The total value at time $t$ is computed as $v_t = b_t + p_t^T h_t$.

---

### 4.2. Model 1: Flat PPO (Memoryless RL Baseline)
**Core Idea:** Standard Proximal Policy Optimization (PPO) using a flat feed-forward Multi-Layer Perceptron (MLP) for both the actor $\pi_\theta$ and critic $V_\phi$ networks.

**Mathematical Formulation:**
The agent relies solely on the current state $s_t$. The advantage $\hat{A}_t$ is estimated using Generalized Advantage Estimation (GAE):
$$ \hat{A}_t = \delta_t + (\gamma \lambda) \delta_{t+1} + \dots + (\gamma \lambda)^{T-t+1} \delta_{T-1} $$
$$ \text{where } \delta_t = r_t + \gamma V_\phi(s_{t+1}) - V_\phi(s_t) $$

The actor network $\pi_\theta(a_t|s_t)$ is updated using the clipped surrogate objective **(Zou et al. 2023, §3.2.3, p. 6)**:
$$ L^{CLIP}(\theta) = \hat{\mathbb{E}}_t \left[ \min\left( \rho_t(\theta)\hat{A}_t, \text{clip}\left(\rho_t(\theta), 1-\epsilon, 1+\epsilon\right)\hat{A}_t \right) \right] $$
where the probability ratio is $\rho_t(\theta) = \frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)}$.

**Reward:** The step-level net return function **(Liu et al. 2024, §3.1, p. 10)**:
$$ r_t = R_t = (b_{t} + p_{t}^T h_{t}) - (b_{t-1} + p_{t-1}^T h_{t-1}) - c_t $$
where $c_t = \delta \sum_{i=1}^N |a_{t,i} - a_{t-1,i}|$ represents the standard proportional transaction cost.

---

### 4.3. Model 2: LSTM-PPO (+ Temporal Memory)
**Core Idea:** Financial data is a Partially Observable MDP (POMDP). We replace the flat MLP with an LSTM to capture temporal dependencies. 

**Mathematical Formulation:**
Instead of a single state $s_t$, the input is a rolling sequence matrix $F_t = [s_{t-W+1}, \dots, s_t]$ over a time window $W$. 
The hidden state $h_t$ and cell state $c_t$ of the LSTM are updated recursively:
$$ h_t, c_t = \text{LSTM}_{\text{cell}}(s_t, h_{t-1}, c_{t-1}) $$
The policy and value functions are now conditioned on the hidden representation: $\pi_\theta(a_t | h_t)$ and $V_\phi(h_t)$.

**Parameters:** Rather than arbitrary tuning, we strictly adopt the architecture validated by **Zou et al. (2023, §4.5.1 & §4.5.2, p. 9)**: Time Window ($W$) = 30, Hidden Size (HS) = 512.

> **📸 [SCREENSHOT PLACEHOLDER 2]** 
> * **Where to capture:** Zou et al. 2023 ("A Novel DRL Based Automated Stock Trading System..."), Page 9.
> * **What to capture:** "Table 2: Comparison of different time windows in LSTM" and "Table 3: Comparison of different hidden sizes of LSTM in PPO", along with the short text stating TW=30 and HS=512 are the best parameters.
> * **Where to paste:** Insert here to justify the direct adoption of the LSTM baseline variables.

**Reward:** Same as Model 1 ($r_t = R_t$).

---

### 4.4. Model 3: Risk-Aware LSTM-PPO (+ Dense Risk-Adjusted Reward)
**Core Idea:** Model 1 and 2 maximize absolute profit, rendering them blind to variance and drawdown risk. Model 3 replaces the raw return reward $r_t$ with the Differential Sharpe Ratio (DSR), providing a step-by-step recursive risk metric.

**Mathematical Formulation:**
The standard Sharpe ratio $S_T = \frac{E[R]}{\sqrt{\text{Var}[R]}}$ requires a full episode to compute, causing sparse delayed rewards. Following **Millea (2021, §5.1.2, p. 8, Eq. 6 & 7)**, we use exponential moving estimates for the first moment ($A_t$) and second moment ($B_t$) of the returns:
$$ A_t = A_{t-1} + \eta (R_t - A_{t-1}) $$
$$ B_t = B_{t-1} + \eta (R_t^2 - B_{t-1}) $$
where $\eta$ is the adaptation rate. Expanding the Sharpe ratio via a Taylor series with respect to $\eta$ yields the DSR step-reward:
$$ D_t = \frac{B_{t-1}\Delta A_t - \frac{1}{2}A_{t-1}\Delta B_t}{(B_{t-1} - A_{t-1}^2 + \epsilon)^{3/2}} $$
*(Note: $\epsilon \approx 1e-8$ is added to the denominator to prevent division-by-zero instability during flat market regimes).* 

For Model 3, the PPO advantage estimation uses $r_t = D_t$ instead of $R_t$.

> **📸 [SCREENSHOT PLACEHOLDER 3]** 
> * **Where to capture:** Millea 2021 ("Deep Reinforcement Learning for Trading—A Critical Survey"), Page 8, Section 5.1.2 "Differential Sharpe Ratio".
> * **What to capture:** Equations 6 and 7 showing the $D_t$ formula and the moving average $A_t$ / $B_t$ updates.
> * **Where to paste:** Insert here to establish the exact mathematical formulation of the dense online reward.

---

### 4.5. Model 4: Friction-Regularized DSR (+ Turnover / Churn Control)
**Core Idea:** DSR mathematically incentivizes the agent to capture tiny, high-Sharpe anomalies, leading to high-frequency action oscillation ("churn"). In live markets, slippage destroys these theoretical returns. Model 4 introduces a friction regularizer directly into the reward formulation.

**Mathematical Formulation:**
To strictly isolate friction-control from risk-sensitivity (M3 → M4 comparison), the turnover penalty must be additive. 
First, we define a return metric that heavily penalizes action shifting (turnover):
$$ \tilde{R}_t = \left( \frac{v_t - v_{t-1}}{v_{t-1}} \right) - c_{slip} \sum_{i=1}^N |a_{t,i} - a_{t-1,i}| $$
where $c_{slip}$ acts as a conservative slippage penalty beyond standard transaction fees. 

The final reward formulation is explicitly regularized:
$$ r_t = D_t - \lambda_{turnover} \cdot \sum_{i=1}^N |a_{t,i} - a_{t-1,i}|^2 $$
where $D_t$ is the DSR calculated from Model 3. Squaring the absolute action shift strictly penalizes violent oscillations (e.g., flipping from $-1$ short to $+1$ long in one step) while permitting smooth rebalancing. This proves mathematically that regularizing action outputs stabilizes the LSTM memory mechanism.

---

## 5. Evaluation Protocol
The four models will be evaluated using identical datasets and walk-forward train/test splits. Overall performance robustness will be tracked systematically across the four standard metrics established in **Huang et al. (2024, "A Self-Rewarding Mechanism in DRL...", §4.2, p. 12)**:

1. **Cumulative Return (CR):** $CR = \frac{P_{end} - P_0}{P_0}$, reflecting the total return of the portfolio at the end of the trading stage.
2. **Annualized Return (AR):** The average level of return presented as an annual percentage.
3. **Sharpe Ratio (SR):** $SR = \frac{\mathbb{E}[R_P] - R_f}{\sigma_P}$, the risk-adjusted return measuring excess return per unit of volatility.
4. **Maximum Drawdown (MDD):** $MDD = \frac{\max(A_x - A_y)}{A_y}$ (where $x>y$ and $A_y > A_x$), reflecting the maximum potential downside risk.

> **📸 [SCREENSHOT PLACEHOLDER 4]** 
> * **Where to capture:** Huang et al. 2024 ("A Self-Rewarding Mechanism..."), Page 12, Section 4.2 "Evaluation Metrics".
> * **What to capture:** The paragraph defining Cumulative Return, Annualized Return, Sharpe Ratio, and Maximum Drawdown.
> * **Where to paste:** Insert here as the final proof of the evaluation baseline methodology.
