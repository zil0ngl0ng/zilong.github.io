---
title: Hypergraph Neural Network(HGNN+)
date: 2022-10-27 10:54:22
categories:
- Deep Learning
tags:
- HGNN
mathjax: true
---
Existing GNN frameworks are deployed based upon simple graphs, which cannot deal with multi-modal/multi-type data. A few hypergraph-based methods have recently been proposed to address the problem by directly concatenating the hypergraphs constructed from each single individual modality/type, which is difficult to learn an adaptive weight for each modality/type. In this paper, we introduce a general high-order multi-modal/multi-type data correlation modeling framework called HGNN+ to learn an optimal representation in a single hypergraph based framework.
<!-- more -->

## Introduction
In our method, hyperedge groups are first constructed to represent latent high-order correlations in each specific modality/type with explicit or implicit graph structures. An adaptive hyperedge group fusion strategy is then used to effectively fuse the correlations from different modalities/types in a unified hypergraph. After that a new hypergraph convolution scheme performed in spatial domain is used to learn a general data representation for various tasks.

So we can get the conclusion that the key points for the HGNN+ are the modeling of hypergraph and propagation mechanism. Here is the figure for proposed hypergraph neural network framework.
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="./HGNN/HGNN+.png" width = "90%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      figure 1: Proposed Hypergraph Neural Network Framework
  	</div>
</center> 

## Hypergraph Modeling
In this part, we introduce the mothod of hypergraph group generation and the combination of hypergraph groups.
### Hyperedge Group Generation
Similar to simple graph, the hypergraph can be modeled as $G(V,E)$, where $V$ represents the set of vertexes and the $E$ represents the set of hyperedge. For generating the hyperedge groups, we can calssify the source data into two calsses, i.e. data with graph structure and data without graph structure.
**Data with Graph Structure**
For data with graph structure, e.g. citation nework and social network, the methods are mainly two aspects, we call it pair-wise edge($E_{pair}$) and k-Hop neighbours($E_{hop}$). The explicit methods are shown in figure $2$.

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="./HGNN/DatawithGraphStructure.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      figure 2: Data with Graph Structure
  	</div>
</center> 

We let $G_s = (V_s, E_s)$ indicate the graph structure, and the matrix $A$ denotes the adjacency matrix of $G_s$. 
$(1)$ For so-called pairwise edge method, $E_{pair}=\{ \{ v_i, v_j\} | (v_i,v_j) \in E_s \}$. Simply say, we replace the edge with hyperedge directly. 
$(2)$ For K-hop neighbours method, the specific process is as follows: for given node $v$, $N_{hop_k} = \{u|A_{uv}^k \ne 0, u \in V_s \}$, so we can get the hyperedge group $E_{hop} = \{N_{hop_k}(v) | v\in V \}$. Simply say, for a given node $v$, we have the parameter $K$, so we think the nodes whose distance to vertex $v$ less than $K$ have the same hyperedge $N_{hop_k}(v)$.
**Data without Graph Structure**
For data without graph structure, we classify the methods into two calsses: using attributes($E_{attribute}$) and using features($E_{feature}$). The methods are shown in figure $3$.

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="./HGNN/DatawithoutGraphStructure.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      figure 3: Data without Graph Structure
  	</div>
</center> 

These methods traget at attributes or features of nodes. 
$(1)$ For using attributes, we mainly focus on the attributes of nodes. For every node $v_i$, it has several attributes $Attr_1,Attr_2,\dots,Attr_k$, so we group the nodes with the same attributes into one hyperedge group. In this way, we can get the hyperedge group $E_{attr}=\{N_{att}(a)|a \in A\}$.
$(2)$ For using features, we may have the features of nodes via embedding layer or others. For node $v_i$ with feature $feature_i$, we can define the distance with two nodes $v_i, v_j$ as $d_{ij} = distance(v_i, v_j)$. Then we have two different methods. The first one is let the $k$ nearest nodes into one group, so we can get the hyperedge group $E_{feature}^{KNN_k} = \{N_{KNN_k}(V)|v \in V\}$. The second method is we classify the nodes whose distance $d_{ij} < d$ into one group. Then we can get the hyperedge group $E_{feature}^{distance_d} = \{N_{dis_d}(v)|v \in V\}$.

### Combination of Hyperedge Groups
Here, several hyperedge groups can be generated using above strategies. Given generated hyperedge groups or natural hyperedge groups, we need to further combine them to generate the final hypergraph. Supposing there are $K$ hyperedge groups $\{E_1, E_2, \dots , E_K\}$, we can have $K$ incidence matrices $H_k \in \{0,1\}^{N \times M_k}$ respectively. And the fusion strategies can be divided into two classes: ***Coequal Fusion*** and ***Adaptive Fusion***.
**Coequal Fusion**
This fusion strategy is simple. And we just concatenate the $K$ incidence matrix $H_k$ directly. So we can get the final hypergraph incidence matrix $H=H_1 || H_2 || H_3 || \dots || H_K$. And the visual illustration of it is shown in figure $4$.

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="./HGNN/CoequalFusion.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      figure 4: Visual Illustration of Coequal Fusion
  	</div>
</center> 

**Adaptive Fusion**
Compared to coequal fusion, adaptive fusion is more complex but can make full use of the multi-modal hybrid high-order correlations. With this atrategy, each hyperedge group is associated with a trainable parameter, which can adaptively adjust the effect of multiple hyperedge groups on the final vertex embeddings. It is defined as:
$$
\left\{\begin{array}{l}
w_k = copy(sigmod(\omega _k), M_k) \\
W = diag(w_1^1,w_1^2,\dots,w_1^{M_1},\dots,w_K^1,\dots,w_K^{M_1})\\
H=W\cdot (H_1 ||H_2 ||\dots ||H_k) 
\end{array}\right.$$
Where $\omega_k \in R$ is a trainable parameter for hyperedge group $E_ K$, vector $w_k = (w_k^1,w_k^2,\dots,w_k^{M_k}) \in R^{M_k}$ denotes the generated weight vector for hyperedge group $k$. And the visual illustraction is shown in figure $5$.

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="./HGNN/AdaptiveFusion.png" width = "50%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      figure 5: Visual Illustration of Adaptive Fusion
  	</div>
</center> 

## Propagation Mechanism
The propagation mechanism is the operation of hyperdege convolution. And the convolution for hypergraph can be reducted in two aspects: spectral and spatial domains.
**Spectral Convolution on Hypergraph**
The convolution in spectral domain can be reducted from spectral graph convolution, with the help of Chebyshv polynomials. Finally the  hyperedge convolution layer HGNNConv can be formulated by:
$$ X^{(l+1)} = \sigma(D_v^{-1/2}HWD_e^{-1}H^TD_v^{-1/2}X^{(l)}\Theta^{(l)}) $$
And $D_v$ and $D_e$ denote the diagonal matrices of vertex degrees and edge degrees respectively.
**General Spatial Convolution on Hypergraph**
Different from convolution in spectral domain, general spatial convolution is more similar to the operation of *Aggragate*. For a given hypergraph $G=\{V,E,W\}$, if we want to update the node $\alpha \in V$, we can aggregate from edge $\beta \in N_e(\alpha)$. For each hyperedge $\beta$, it has vertex inter-neighbor set $N_v(\beta)$, so we can make use of the info in $N_v(\beta)$ to update the hyperedge feature. After the definiton and reduction, the matrix format of $HGNNConv+$ can be written as: 
$$ X^{(t+1)} = \sigma(D_v^{-1}HWD_e^{-1}H^TX^{(t)}\Theta^{(t)}) $$

+ <u>Paper: Hypergraph Neural Networks</u>
+ <u>Paper: HGNN⁺: General Hypergraph Neural Networks</u>