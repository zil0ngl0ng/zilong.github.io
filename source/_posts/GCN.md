---
title: Graph Convolution Network(GCN)
date: 2022-10-20
categories:
- Deep Learning
tags:
- GCN
---

This article is mainly about the backgrounds of GCN and its reduction from spectral graph convolution.

<!--more-->

## Introduction
Consider the problem of classfying nodes in a graph, the labels are only available for a small subset of nodes. So this problem can be framed as graph-based semi-supervised learning. In past, we may use a Laplacian regularization term in the loss function and the label information is smoothed over the graph. However, this might restrict modeling capacity, as graph edges need not necessarily encode node similarity, but could contain additional information.

In this paper, the author encode the graph structure directly using a neural network model $f(X, A)$ and train on a supervised target $L0$ for all nodes with labels, thereby avoiding explicit graph-based regularization in the loss function.  Conditioning $f(·)$ on the adjacency matrix of the graph will allow the model to distribute gradient information from the supervised loss L0 and will enable it to learn representations of nodes both with and without labels.

## Reduction
In this part, I motivate the form of propagation rule via a first-order approximation of localized spectral filters on graphs.

For signal $ x \in R^N $



