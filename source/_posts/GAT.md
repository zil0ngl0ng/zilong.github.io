---
title: Graph Attention Network(GAT)
date: 2022-10-25 09:52:40
categories:
- Deep Learning
tags:
- GAT
mathjax: true
---

For graph structure problems, there exist two kinds of approaches, spectral and non-spectral approaches.For spectral approaches, the learned filters depend on the Laplacian eigenbasis, which depends on the graph structure. Thus, a model trained on a specific structure can not be directly applied to a graph with a different structure. For non-spectral approaches, one of the challenges is to define an operator which works with different sized neighborhoods and maintains the weight sharing property. So the author introduce the **Graph attention network(GAT)**, computing the hidden representations of each node in the graph by attending over its neighbors and can be directly applicable to inductive learning problems.

<!-- more -->

## GAT ARCHITECTURE
### Graph Attention Layer
For a single graph attention layer, the input is a set of node features, $ h=\{ \vec{h}_1,\vec{h}_2,\dots,\vec{h}_N \},\vec{h}_i \in R^F $, where $ N $ is the number of nodes and $ F $ is the dimension of node feature. The layer produce a new set of node features $ h'=\{ \vec{h}'_1,\vec{h}'_2, \dots, \vec{h}'_N  \}, \vec{h}'_i \in R^{F'} $. 

In order to obtain sufficient expressive power to transform the feature to higher-level features, we first use a shared linear transformation $W \in R^{F \times F'}$, so $\vec{h}_i \to W\vec{h}_i$. We define a shared attentional mechanism to compute the attention coefficients: $$ e_{ij} = \alpha (W\vec{h}_i,W\vec{h}_j) \tag{1}$$
in which, $ e_{ij} $ indicates the importance of node $j$ to $i$. In graph structure, we use masked attention——only compute $ e_{ij} $ for node $ j \in N_i $, $N_i$ is the neighbourhood of node $i$. In order to make coefficients easily comparable across different nodes, we normalize them across all choices of $j$ using the softmax function:
$$ \alpha_{ij}=softmax_j(e_{ij})=\frac{exp(e_{ij})}{\sum_{k \in N_i} exp(e_{ik})}  \tag{2}$$

In this paper, $\alpha$ is a single-layer feedforward neural network parametrized by a weight vector $ \vec a \in R^{2F'} $ and applying the $LeakyReLU$ nonlinearity. So the formulation $(1)$ equals to:
$$ e_{ij} = \alpha (W\vec{h}_i,W\vec{h}_j)=LeakyReLU(\vec a^T [W\vec{h}_i || W\vec{h}_j]) \tag{3}$$

Once obtained, the normalized attention coefficients are used to compute a linear combination of the features corresponding to them, to serve as the final output features for every node:
$$ \vec{h}'_i=\sigma(\sum_{j \in N_i} \alpha_{ij}W\vec{h}_j) \tag{4}$$

For multi-head attention, if $K$ independent attention mechanisms are used, the formulation is shown as $(5)$:
$$ \vec h'_i = ||_{k=1}^{K} \sigma(\sum_{j \in N_i} \alpha_{ij}^kW^k\vec h_j) \tag{5}$$
If it's the final layer of network, concatenation is no longer sensible, so it need to be justified as follows:
$$ \vec h'_i =  \sigma(\frac{1}{K}\sum_{k=1}^{K} \sum_{j \in N_i} \alpha_{ij}^kW^k\vec h_j) \tag{5}$$

It's worthy to notice that, when compute the matrix of $e_{ij}^0 = \vec a^T [W\vec{h}_i || W\vec{h}_j]$, a trick is used and the formulation is $[e_{ij}^0] = W^ha_1^T + (W^ha_2^T)^T$, as $a_1^T,a_2^T$ are parts of $a^T$. And the reduction is followed.
> Reduction of **$[e_{ij}^0] = W^ha_1^T + (W^ha_2^T)^T$**
&ensp; $\begin{aligned}
e_{ij}^0 &= [W\vec{h}_i || W\vec{h}_j] \vec a^T \\
&= [W\vec{h}_i || W\vec{h}_j] [\vec a_1 || \vec a_2]^T ,\vec a_1=\vec a(0:F',:),\vec a_2=\vec a(F'+1:2F',:) \\
&= W\vec{h}_i\vec a_1 + W\vec{h}_j\vec a_2 \\
[e_{ij}^0] &= [W\vec{h}_i\vec a_1 + W\vec{h}_j\vec a_2] \\
&=\begin{bmatrix}
 W\vec{h}_1\vec a_1 + W\vec{h}_1\vec a_2 & W\vec{h}_1\vec a_1 + W\vec{h}_2\vec a_2 & \dots &W\vec{h}_1\vec a_1 + W\vec{h}_N\vec a_2\\
 W\vec{h}_2\vec a_1 + W\vec{h}_1\vec a_2 & W\vec{h}_2\vec a_1 + W\vec{h}_2\vec a_2 & \dots &W\vec{h}_2\vec a_1 + W\vec{h}_N\vec a_2\\
 \vdots & \vdots & \ddots & \vdots \\
 W\vec{h}_N\vec a_1 + W\vec{h}_1\vec a_2 & W\vec{h}_N\vec a_1 + W\vec{h}_2\vec a_2 & \dots &W\vec{h}_N\vec a_1 + W\vec{h}_N\vec a_2
\end{bmatrix}\\
&=\begin{bmatrix}
 W\vec{h}_1\vec a_1 & W\vec{h}_1\vec a_1 & \dots & W\vec{h}_1\vec a_1 \\
 W\vec{h}_2\vec a_1 & W\vec{h}_2\vec a_1 & \dots & W\vec{h}_2\vec a_1 \\
 \vdots & \vdots &  & \vdots \\
 W\vec{h}_N\vec a_1 & W\vec{h}_N\vec a_1 & \dots & W\vec{h}_N\vec a_1
\end{bmatrix} + \begin{bmatrix}
 W\vec{h}_1\vec a_2 & W\vec{h}_2\vec a_2 & \dots & W\vec{h}_N\vec a_2 \\
 W\vec{h}_1\vec a_2 & W\vec{h}_2\vec a_2 & \dots & W\vec{h}_N\vec a_2 \\
 \vdots & \vdots &  & \\
 W\vec{h}_1\vec a_2 & W\vec{h}_2\vec a_2 & \dots & W\vec{h}_N\vec a_2
\end{bmatrix}\\
&=\begin{bmatrix}
W\vec{h}_1 \\
\vdots \\
W\vec{h}_N
\end{bmatrix}\vec{a_1} + \begin{bmatrix}\begin{bmatrix}
W\vec{h}_1 \\
\vdots \\
W\vec{h}_N
\end{bmatrix}\vec{a_2}\end{bmatrix}^T \\
&=W\vec{h}\vec{a}_1 + (W\vec{h}\vec{a}_2)^T
\end{aligned}$