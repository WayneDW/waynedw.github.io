---
title: 'Autoregressive Flow'
subtitle: Efficient transport maps
date: 2024-03-16
permalink: /posts/autoregressive_flow/
category: Flow
---

$$\begin{align}
  .
\end{align}$$

### Normalizing Flows

Normalizing flows are early pioneers in generating modeling through a series of invertible transformations. The main idea is to transform a random vector $u$ through a transport map $\mathcal{T}$:

$$\begin{align}
  \mathrm{x}=\mathcal{T}(u), \text{where}\  u\sim p_u(u).
\end{align}$$

If $\mathcal{T}$ is a bijective, differentiable function, the density of $x$ can be obtained by a change of variables

$$\begin{align}
  p_x(x)=p_u(u)\bigg|\text{det} J_{\mathcal{T}}(u) \bigg|^{-1}.
\end{align}$$

where $\text{det} J_{\mathcal{T}}(u)$ quantifies the change of volume w.r.t. u, such that $\text{det} J_{\mathcal{T}}(u)\approx \frac{\text{Vol}(\mathrm{d} x)}{\text{Vol}(\mathrm{d} u)}$. 




It is also equivalent to the Jacobian of $\mathbb{T}^{-1}$ in a way that.


$$\begin{align}
  p_x(x)=p_u(\mathcal{T}^{-1}(X))\bigg|\text{det} J_{\mathcal{T}^{-1}}(u) \bigg|.
\end{align}$$

Empirically, the transport map $\mathcal{T}$ can be implemented with a neural network.


#### Composition Property

One such transformation is not sufficient to model complex distributions. It is standard to conduct multiple transformations $\mathcal{T}_1$, $\mathcal{T}_2, \cdots$, $\mathcal{T}_K$ such that $\mathcal{T}=\mathcal{T}_K \circ \cdots \circ \mathcal{T}_2 \circ \mathcal{T}_1$. If $\\{ \mathcal{T}_k \\}_1^K$ are also bijective and invertible, we have

$$\begin{align}
  &\mathcal{T}^{-1}=(\mathcal{T}_K \circ \cdots \circ \mathcal{T}_2 \circ \mathcal{T}_1)^{-1}=\mathcal{T}_1^{-1} \circ \mathcal{T}_2^{-1} \circ \dots \mathcal{T}_K^{-1}.\notag \\
  & \text{det} J_{\mathcal{T}}(u) = \text{det} J_{\mathcal{T}_K \circ \dots \mathcal{T}_1}(u) = \text{det} J_{\mathcal{T}_1}(u)  \cdot \text{det} J_{\mathcal{T}_2}(\mathcal{T}_1(u))  \cdot \text{det} J_{\mathcal{T}_K}(\mathcal{T}_{K-1}(\cdots(\mathcal{T}_1(u)))).
\end{align}$$


#### KL Divergence


Denote by  $\phi$ and $\psi$ the parameters of $\mathcal{T}$ and $p_u(u)$, respectively. The forward KL divergence follows that

$$\begin{align}
  \mathcal{L}(\phi, \psi)&=\text{KL}(p^{\star}_{X}(x)\| p_X(x; \phi, \psi)) \notag \\
  &=-\mathbb{E}_{p^{\star}_{X}(x)} [\log p_X(x; \phi, \psi)] + \text{const} \notag \\
  &=-\mathbb{E}_{p^{\star}_{X}(x)} [\log p_u(\mathcal{T}^{-1}(x; \phi); \psi) + \log |\text{det} J_{\mathcal{T}^{-1}}(x; \phi)|]+\text{const}. \notag
\end{align}$$

The forward KL divergence is well-suited for situations in which we have samples from the
target distribution (or the ability to generate them), but we cannot necessarily evaluate
the target density $p^{\star}_X(x)$ (polish words ). 


Alternatively, when we are interested to evaluate model density $p_X^{\star}(x)$, we can try to optimize the reverse KL divergence 

$$\begin{align}
  \mathcal{L}(\phi, \psi)&=\text{KL}(p_X(x; \phi, \psi) \| p^{\star}_{X}(x)) \notag \\
  &=\mathbb{E}_{p_{X}(x; \phi, \psi)} [\log p_X(x; \phi, \psi) - \log p_{X}^{\star}(x)]  \notag \\
  &=-\mathbb{E}_{p_{u}(u; \psi)} [\log p_u(u; \psi) - \log |\text{det} J_{\mathcal{T}}(u; \phi)| - \log p_{X}^{\star}(\mathcal{T}(u; \phi))]. \notag
\end{align}$$

In empirical training, we resrot to Monte Carlo samples to approximate the expectation.


### Autogressive Flows

Autoregressive flows were one of the earliest flows developed. We can transform any distribution $p_x(x)$ into a uniform distribution in $(0, 1)^D$ using maps with a triangular Jacobian. (polish)

To achieve the goal of universal representation, we leverage conditional probability and decompose $p_X(x)$ into a product of conditional probabilities as follows

$$\begin{align}
  p_X(x) = \Pi_{i=1}^D p_X(x_i | x_{<i}).
\end{align}$$

Define the transformation $F$ to be the CDF function of the conditional density:

$$\begin{align}
  z_i = F_i(x_i, x_{<i})=\int_{-\infty}^{x_i} p_X(x_i'|x_{<i}) \mathrm{d} x_i'=\text{P}(x_i'\leq x_i | x_{<i}).
\end{align}$$

The invertibility of $F$ leads to 

$$\begin{align}
  x_i = (F_i(\cdot, x_{<i}))^{-1} (z_i) \text{ for } i=1,2,..., D.
\end{align}$$

Notably, the Jacobian of $F$ is a lower trianagular matrix and the determinant of $F$ is equal to the product of diagonal elemants.

$$\begin{align}
  \text{det} J_F(x) = \Pi_{i=1}^D \frac{\partial F_i}{\partial x_i} = \Pi_{i=1}^D p_X(x_i | x_{<i})=p_X(x).
\end{align}$$



#### Masked Autoencoder

The iterative conditional operations appear to be natural, but the challenge is how to model the conditional probability efficiently. We can employ a na\"ive RNN encoder to model in a sequential manner. However, it may not scale well to high dimensions. To tackle this issue, a mind-blowing idea is to work on a vanilla autoencoder with clear masks to satisfy the autoregressive property in an efficient parallel scheme.  Since output $\tilde x_d$ must depend only on the preceding inputs $x_{<d}$, it means that there must be no computational path between output unit $\tilde x_d$ and any of the input units $x_d, \cdots, x_D$.

#### Masked AR

#### Inverse AR




understand the loss function:

zs, prior_logprob, log_det = model(x)
logprob = prior_logprob + log_det
loss = -torch.sum(logprob) # NLL

Masked autoencoder (why do we need parity?) RNN is not needed, better efficiency. https://www.youtube.com/watch?v=lNW8T0W-xeE




JAX: Real-NVP --  https://blog.evjang.com/2019/07/nf-jax.html

JAx: Flow: https://github.com/ChrisWaites/jax-flows

https://github.com/karpathy/pytorch-normalizing-flows/tree/master

FYI