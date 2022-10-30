---
title: Graph Convolution Network(GCN)
date: 2022-10-20
categories:
- Deep Learning
tags:
- GCN
mathjax: true
---

This article is mainly about the backgrounds of GCN and its reduction from spectral graph convolution.

<!--more-->

## Introduction
Consider the problem of classfying nodes in a graph, the labels are only available for a small subset of nodes. So this problem can be framed as graph-based semi-supervised learning. In past, we may use a Laplacian regularization term in the loss function and the label information is smoothed over the graph. However, this might restrict modeling capacity, as graph edges need not necessarily encode node similarity, but could contain additional information.

In this paper, the author encode the graph structure directly using a neural network model $f(X, A)$ and train on a supervised target $L0$ for all nodes with labels, thereby avoiding explicit graph-based regularization in the loss function.  Conditioning $f(·)$ on the adjacency matrix of the graph will allow the model to distribute gradient information from the supervised loss L0 and will enable it to learn representations of nodes both with and without labels.

## Reduction
In this part, I motivate the form of propagation rule via a first-order approximation of localized spectral filters on graphs.

For signal $ x \in R^N $, we define the spectral convolutions on graph as 
$$ g_{\theta } * x = Ug_{\theta }U^Tx \tag{1}$$ 
in which $ U $ is the matrix of eigenvectors of normalized graph Laplacian $ L=I_N - D^{-1/2}AD^{-1/2}=U\Lambda U^{-1}=U\Lambda U^T $, $ g_{\theta } $ can be understand as the function of $ \Lambda $(eigenvalues of $ L $). It's worth to notice that for a real symmetric matrix L's EVD matrix is orthogonal, so $U^{-1} = U^T$.

With the help of Chebyshev polynomials $ T_k(x) $ up to $ K $ order:
$$ g_{\theta'}(\Lambda) \approx \sum_{k=0}^{K} {\theta'_kT_k(\tilde{\Lambda})}  \tag{2}$$ 
in which, $ \tilde{\Lambda}=\frac{2}{\lambda_{max}}\Lambda-I_N, \theta' \in R^k $ is the vector of Chebyshev coefficients. And the Chebyshev polynomials are recursively defined as $ T_k(x)=2xT_{k-1}(x)-T_{k-2}(x), T_0(x)=1, T_1(x)=x. $

Combine the formulation $(1)$ and $(2)$, we can know that 
$$ g_{\theta'} * x \approx \sum_{k=0}^{K}{\theta'_k T_k(\tilde{L})x} \tag{3}$$
in which, $ \tilde L = \frac{2}{\lambda_{max}}L-I_N $, and all the reduction process is following.
> **The Reduction Process**
$\begin{aligned}
g_{\theta } * x &= Ug_{\theta }U^Tx\\
& \approx U \sum_{k=0}^{K} {\theta'_kT_k(\tilde{\Lambda})} U^Tx\\
&= \sum_{k=0}^{K} {\theta'_k UT_k(\tilde{\Lambda})} U^Tx \\
&= \sum_{k=0}^{K} {\theta'_k T_k(U\tilde{\Lambda} U^T)} x \\
&= \sum_{k=0}^{K} {\theta'_k T_k(\tilde{L})} x 
\end{aligned} $ 
> * **$ UT_k(\tilde{\Lambda}) U^T = T_k(U\tilde{\Lambda} U^T) $**
(mathematical induction)
① for $k=0$, $T_0(\tilde{\Lambda})=1=T_0(U\tilde{\Lambda} U^T)$
② assume that for $k \le m$, the equation is correct
③ IF $k=m+1$, 
&ensp; $\begin{aligned} 
UT_{m+1}(\tilde{\Lambda}) U^T &=  U(2\tilde{\Lambda}T_{m}(\tilde{\Lambda})-T_{m-1}(\tilde{\Lambda}))U^T \\
&= 2U\tilde{\Lambda}T_{m}(\tilde{\Lambda})U^T - U T_{m-1}(\tilde{\Lambda})U^T \\
&= 2U\tilde{\Lambda}U^TUT_{m}(\tilde{\Lambda})U^T - U T_{m-1}(\tilde{\Lambda})U^T \\
&= 2(U\tilde{\Lambda}U^T)T_{m}(U\tilde{\Lambda}U^T) - T_{m-1}(U\tilde{\Lambda}U^T) \\
&= T_{m+1}(U\tilde{\Lambda} U^T) 
\end{aligned}$
> * **$ U\tilde{\Lambda}U^T=\tilde{L} $**
$\begin{aligned}  
U\tilde{\Lambda}U^T &= U(\frac{2}{\lambda_{max}}\Lambda-I_N)U^T \\
&=\frac{2}{\lambda_{max}}U\Lambda U^T-U I_N U^T \\
&=\frac{2}{\lambda_{max}}L - I_N \\
&=\tilde{L}
\end{aligned}$

After approximating the spectral graph convolution formulation, we need to further simplyfy it. We have known the formulation:
$$ g_{\theta'} * x \approx \sum_{k=0}^{K}{\theta'_k T_k(\tilde{L})x} \tag{4}$$

In order to further simplyfy it, we let $ K=1,\lambda_{max}=2 $, so the formulation$(4)$ can be simplyfied as:
$$\begin{aligned}
g_{\theta'} * x &\approx \sum_{k=0}^{K}{\theta'_k T_k(\tilde{L})x} \\
&=\theta'_0 T_0(\tilde{L})x + \theta'_1 T_1(\tilde{L})x \\
&=\theta'_0 x + \theta'_1 (L-I_N)x \\
&=\theta'_0 x - \theta'_1D^{-1/2}AD^{-1/2}x
\end{aligned}$$

Then we let $ \theta = \theta'_0 = -\theta'_1 $, so formulation can be expressed as:
$$ g_{\theta'} * x \approx \theta(I_N+D^{-1/2}AD^{-1/2})x \tag{5}$$

Note that $I_N+D^{-1/2}AD^{-1/2}$ has eigenvalues in the range [0, 2]. Repeated application of this operator can therefore lead to numerical instabilities and exploding/vanishing gradients when used in a deep neural network model. To alleviate this problem, we introduce the following renormalization trick $ I_N+D^{-1/2}AD^{-1/2} \longrightarrow \tilde{D}^{-1/2}\tilde{A} \tilde{D}^{-1/2}$ , with $\tilde{A} = A+I_N, \tilde{D}_{ii}=\sum_{j}A_{ij}$.

After using the trick, formulation $(6)$ can be finally expressed as 
$$ g_{\theta'} * x \approx \theta \tilde{D}^{-1/2}\tilde{A} \tilde{D}^{-1/2}x,x \in R^N \tag{6}$$

To extend the $ x \in R^N $ to $ x \in R^{N \times F} $, we can express as the form of matrix, so finally we can reduct the propagation rule as formulation $(7)$:
$$
Z=\tilde{D}^{-1/2} \tilde{A}\tilde{D}^{-1/2}X\Theta,\Theta \in R^{C \times F},X \in R^{N \times C} \tag{7}
$$

For example, for a 2-layers GCN models, if we want to finish the classfication task, the propagation rule can be expressed as follows:
$$ Z=softmax(\hat{A}Relu(\hat{A}XW^{0})W^{(1)}) $$

+ <u>Paper: Semi-Supervised Classification with Graph Convolutional Networks</u>