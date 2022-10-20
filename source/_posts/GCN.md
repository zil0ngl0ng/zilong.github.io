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
in which $ U $ is the matrix of eigenvectors of normalized graph Laplacian $ L=I_N - D^{-1/2}AD^{-1/2}=U\Lambda U^T $, $ g_{\theta } $ can be understand as the function of $ \Lambda $(eigenvalues of $ L $).

With the help of Chebyshev polynomials $ T_k(x) $ up to $ K $ order:
$$ g_{\theta'}(\Lambda) \approx \sum_{k=0}^{K} {\theta'_kT_k(\tilde{\Lambda})}  \tag{2}$$ 
in which, $ \tilde{\Lambda}=\frac{2}{\lambda_{max}}\Lambda-I_N, \theta' \in R^k $ is the vector of Chebyshev coefficients. And the Chebyshev polynomials are recursively defined as $ T_k(x)=2xT_{k-1}(x)-T_{k-2}(x), T_0(x)=1, T_1(x)=x. $

Combine the formulation $(1)$ and $(2)$, we can know that 
$$ g_{\theta'} * x \approx \sum_{k=0}^{K}{\theta'_k T_k(\tilde{L})x} $$
in which, $ \tilde L = \frac{2}{\lambda_{max}}L-I_N $, and all the reduction process is following.
> The Reduction Process

