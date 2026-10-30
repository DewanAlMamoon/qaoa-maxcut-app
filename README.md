# qaoa-maxcut-app
QAOA algorithm implementation with max-cut.

---

# QAOA Max-Cut Solver

An open-source, modular framework for solving combinatorial optimization problems using the **Quantum Approximate Optimization Algorithm (QAOA)**, specifically targeting the **Max-Cut** graph problem.

---

## 1. Introduction & Executive Summary

Combinatorial optimization lies at the core of computational complexity theory, logistics, operational research, and network theory. Many central optimization problems—including Max-Cut—belong to the **NP-hard** complexity class, rendering exact classical solutions computationally intractable for large-scale instances.

The **Quantum Approximate Optimization Algorithm (QAOA)**, introduced by Farhi, Goldstone, and Gutmann (2014), is a variational quantum-classical hybrid algorithm designed for Near-Term Intermediate-Scale Quantum (**NISQ**) devices. QAOA maps discrete optimization problems onto spin-glass Hamiltonians and iteratively optimizes variational parameters $(\boldsymbol{\gamma}, \boldsymbol{\beta})$ using a classical feedback loop to prepare ground (or near-ground) quantum states.

---

## 2. Problem Formulation: Max-Cut to Ising Hamiltonian

### 2.1 The Max-Cut Problem

Let $G = (V, E)$ be an undirected, unweighted graph with $\vert{}V\vert{} = n$ vertices and $\vert{}E\vert{} = m$ edges. The goal of the Max-Cut problem is to partition $V$ into two disjoint subsets $S$ and $V \setminus S$ such that the number of edges spanning across the partition is maximized.

Assign a binary decision variable $s_i \in \{-1, +1\}$ to each vertex $i \in V$, where $s_i = +1$ denotes $i \in S$ and $s_i = -1$ denotes $i \in V \setminus S$. The classical cost function to maximize is:

$$C(\boldsymbol{s}) = \frac{1}{2} \sum_{(i,j) \in E} (1 - s_i s_j)$$

### 2.2 Quantum Mapping (Cost Hamiltonian)

To map the problem to quantum hardware, binary variables $s_i$ are promoted to Pauli-$Z$ operators acting on $n$ qubits:

$$\hat{s}_i \longrightarrow \hat{Z}_i$$

The classical cost function is mapped to a diagonal **Cost Hamiltonian** $\hat{H}_C$:

$$\hat{H}_C = \frac{1}{2} \sum_{(i,j) \in E} \left( \hat{I} - \hat{Z}_i \hat{Z}_j \right)$$

Finding the maximum cut corresponds to finding the ground state $\vert{}\psi_0\rangle$ that minimizes $\langle \hat{H}_C \rangle$ (or maximizes $\langle C \rangle$).

---

## 3. Mathematical & Theoretical Framework of QAOA

QAOA prepares a parameterized quantum state $\vert{}\boldsymbol{\gamma}, \boldsymbol{\beta}\rangle$ by alternating applications of two non-commuting Hamiltonians: the **Cost Hamiltonian** $\hat{H}_C$ and the **Mixer Hamiltonian** $\hat{H}_B$.

### 3.1 Initial State Preparation

The system starts in an equal superposition of all computational basis states, which is the ground state of the Mixer Hamiltonian:

$$\vert{}\psi_0\rangle = \vert{}+\rangle^{\otimes n} = \frac{1}{\sqrt{2^n}} \sum_{x \in \{0,1\}^n} \vert{}x\rangle = \bigotimes_{i=1}^n \frac{\vert{}0\rangle + \vert{}1\rangle}{\sqrt{2}}$$

### 3.2 The Mixer Hamiltonian

The standard transverse-field mixer operator is defined as:

$$\hat{H}_B = \sum_{i \in V} \hat{X}_i$$

### 3.3 Parametric Quantum Ansatz Structure

For a depth-$p$ QAOA circuit, we apply $p$ layers of parameterized unitaries specified by angles $\boldsymbol{\gamma} = (\gamma_1, \dots, \gamma_p)$ and $\boldsymbol{\beta} = (\beta_1, \dots, \beta_p)$:

$$\vert{}\boldsymbol{\gamma}, \boldsymbol{\beta}\rangle = \left( \prod_{l=1}^p \hat{U}_B(\beta_l) \hat{U}_C(\gamma_l) \right) \vert{}\psi_0\rangle$$

Where the unitary operators are given by matrix exponentials:

* **Problem Unitary:**

$$\hat{U}_C(\gamma_l) = e^{-i \gamma_l \hat{H}_C} = \prod_{(i,j) \in E} \exp\left( -i \frac{\gamma_l}{2} (\hat{I} - \hat{Z}_i \hat{Z}_j) \right)$$


* **Mixer Unitary:**

$$\hat{U}_B(\beta_l) = e^{-i \beta_l \hat{H}_B} = \prod_{i \in V} \exp\left( -i \beta_l \hat{X}_i \right)$$



### 3.4 Classical Optimization Loop

The quantum processing unit (QPU) evaluates the expectation value:

$$E(\boldsymbol{\gamma}, \boldsymbol{\beta}) = \langle \boldsymbol{\gamma}, \boldsymbol{\beta} \vert{} \hat{H}_C \vert{} \boldsymbol{\gamma}, \boldsymbol{\beta} \rangle$$

A classical optimization algorithm (e.g., COBYLA, SPSA, ADAM, or Nelder-Mead) processes measurement outcomes and updates parameters to minimize $E(\boldsymbol{\gamma}, \boldsymbol{\beta})$:

$$
(\boldsymbol{\gamma}^*, \boldsymbol{\beta}^*) = \arg\min_{\boldsymbol{\gamma}, \boldsymbol{\beta}} E(\boldsymbol{\gamma}, \boldsymbol{\beta})
$$

---

## 4. Quantum Gate Decomposition & Physical Implementation

To execute $\hat{U}_C(\gamma)$ and $\hat{U}_B(\beta)$ on gate-based quantum devices, continuous Hamiltonians are mapped to native hardware gates:

1. **Mixer Layer $e^{-i \beta \hat{X}_i}$:** Implemented on each qubit using single-qubit rotation gates:

$$R_X(2\beta) = \begin{pmatrix} \cos\beta & -i\sin\beta \\ -i\sin\beta & \cos\beta \end{pmatrix}$$


2. **Cost Layer $e^{-i \frac{\gamma}{2} (\hat{I} - \hat{Z}_i \hat{Z}_j)}$:** Implemented using two-qubit entangling gates. Ignoring global phase factors, $e^{i \gamma \hat{Z}_i \hat{Z}_j / 2}$ decomposes into standard $\text{CNOT}$ and $R_Z$ gates:

$$
\text{CNOT}_{i,j} \longrightarrow R_Z(2\gamma)_j \longrightarrow \text{CNOT}_{i,j}
$$



```text
Qubit i: ───■───────────────────■───
            │                   │   
Qubit j: ───X───[ Rz(2γ) ]──────X───

```

---

## 5. Applications & Complexity Analysis

* **Approximation Guarantees:** For $p \to \infty$, QAOA converges to the exact ground state by the Adiabatic Theorem. At $p=1$, QAOA guarantees an approximation ratio $\ge 0.6924$ on 3-regular graphs, outperforming random assignment ($0.5$).
* **Applications:** Network layout optimization, image segmentation, portfolio optimization, clustering analysis, VLSI circuit design, and scheduling problems.

---
