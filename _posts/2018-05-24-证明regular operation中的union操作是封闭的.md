---
published: true
layout: post
subtitle: 证明regular operation中的union操作是封闭的
author: persuez
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
  - 计算理论 DFA
---
# 证明思路
$M_1$和$M_2$是两个DFA，且$L(M_1)=A,L(M_2)=A_2$，现在要证明存在一个DFA$M$可以recognizes $A \cup A_2$(我们将会找到这样一个DFA，而不仅仅是存在)，即$L(M)=A \cup A_2$。我们找这样一个DFA的方法是通过模拟simulate。但是在读到一个字符时，我们的DFA要往哪个状态转移呢？DFA不允许我们遇到一个字符可以有多条路走(所以之后我们用NFA证明会很简单)，那怎么办呢？这时候，聪明的人用二元组的方式表示转移状态，即$(r_i,r_j)$，其中，$r_i \in Q_1，r_j \in Q_2$。这样子，$M$的状态集$Q=Q_1 × Q_2$，'$×$'表示笛卡尔积。那接下来就容易证明了。
# 证明
设$L(M_1) = A_1, M_1 = (Q_1, \sum_1, \delta_1, q_1, F_1); L(M_2) = A_2, M_2 = (Q_2, \sum_2, \delta_2, q_2, F_2)$,我们构造$M = (Q, \sum, \delta, q_0, F)$，使得$L(M) = A_1 \cup A_2$,其中：
1. $Q = \lbrace (r_1, r_2) \mid r_1 \in Q_1\ and\ r_2 \in Q_2 \rbrace$
2. 这里我们假设$\sum = \sum_1 = \sum_2$，如果$\sum_1 \neq \sum_2$,我们只要使$\sum = \sum_1 \cup \sum_2$,并$Q_1 = Q_1 \cup r_{error1}$,$Q_2 = Q_2 \cup r_{error2}$,$\delta_1(r, a) = r_{error1}$, $r \in Q_1$, $a \notin \sum_1$,$\delta_2(r, a) = r_{error2}$, $r \in Q_2$, $a \notin \sum_2$
3. $\delta(r_1,r_2), a) = (\delta_1(r_1, a), \delta_2(r_2, a))$, 其中$(r_1, r_2) \in Q\ and\ a \in \sum$
4. $q_0 = (q_1, q_2)$
5. $F=\lbrace (r_1, r_2) \mid r_1 \in F_1\ or\ r_2 \in F_2 \rbrace$
证毕。
因为上面证明是简单直白的，所以该证明的正确性读者只需模拟simulate上述过程就知道了，不再证明其正确性。
