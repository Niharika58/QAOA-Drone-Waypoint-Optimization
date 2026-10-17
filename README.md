# QAOA-Based Drone Waypoint Optimization: Processor A vs. Processor B

## 1. Project Overview

The objective of this experiment is to investigate how quantum optimization techniques can support drone inspection planning under limited battery capacity or flight-energy constraints.

A drone may need to inspect multiple locations, such as disaster-affected areas, infrastructure assets, or potential threat zones. Each location has a priority value, while visiting it incurs an energy cost. Since the drone has a limited energy budget, it cannot necessarily inspect every location during a single mission.

To address this problem, we formulated drone waypoint selection as a 0/1 Knapsack Optimization Problem and explored a QAOA-inspired quantum circuit for two simulated processor configurations:

- **Processor A:** 5 qubits representing 5 candidate inspection sites.
- **Processor B:** 9 qubits representing 9 candidate inspection sites.

Both configurations used the same energy budget of 10 units. We compared their selected waypoints, total priority, energy consumption, optimization results, and execution time.

The purpose was to study the trade-off between the number of candidate locations, solution quality, and computational cost.

## 2. Problem Formulation

We model the inspection-planning problem using the following parameters.

| Symbol | Meaning |
|---|---|
| $n$ | Number of candidate inspection sites |
| $x_i$ | Binary decision variable for site $i$ |
| $v_i$ | Priority or value of site $i$ |
| $w_i$ | Energy required to inspect site $i$ |
| $B$ | Maximum available energy budget |

Each decision variable is binary:

$$x_i \in \{0,1\}$$

- $x_i=1$: The drone selects site $i$.
- $x_i=0$: The drone does not select site $i$.

The goal is to maximize the total priority of the selected sites without exceeding the available energy.

### Objective function

$$\boxed{\max \sum_{i=1}^{n}v_i x_i}$$

This function calculates the total priority of all selected inspection sites.

### Energy constraint

$$\boxed{\sum_{i=1}^{n}w_i x_i \leq B}$$

This constraint ensures that the total energy required by the selected sites does not exceed the drone's available energy budget.

### Complete optimization model

$$
\begin{aligned}
\text{Maximize:}\quad & \sum_{i=1}^{n}v_i x_i\\
\text{Subject to:}\quad & \sum_{i=1}^{n}w_i x_i\leq B\\
& x_i\in\{0,1\},\quad i=1,\ldots,n
\end{aligned}
$$

This mathematical formulation allows the system to select the most valuable feasible combination of inspection sites.

## 3. Experimental Configuration

We evaluated two simulated configurations using the following datasets.

### Processor A (5 qubits)

Five candidate inspection sites:

- Priorities: $9,7,6,5,4$
- Energy costs: $6,5,4,3,2$
- Energy budget: 10 units

### Processor B (9 qubits)

Nine candidate inspection sites:

- Priorities: $9,7,6,5,4,8,3,6,5$
- Energy costs: $6,5,4,3,2,5,2,4,3$
- Energy budget: 10 units

### Shared QAOA settings

Both configurations used the same principal QAOA settings:

| Parameter | Setting |
|---|---|
| QAOA layers ($p$) | 2 |
| Classical optimizer | COBYLA |
| Maximum optimization iterations | 60 |
| Sampling shots | 4,096 |
| Random seed | 42 |

The experiments were executed in a classical simulation environment. The labels Processor A and Processor B refer to the two simulated qubit configurations, not two physical quantum processors.

## 4. Algorithms and Mathematical Formulas Used

### 4.1 0/1 Knapsack optimization

The knapsack formulation is the mathematical model that defines the problem. It determines which sites should be selected to maximize priority while satisfying the energy constraint.

For example, Processor A selected sites B, D, and E:

$$
\begin{aligned}
\text{Priority} &= 7+5+4=16\\
\text{Energy} &= 5+3+2=10
\end{aligned}
$$

The selected combination achieves a priority of 16 while using the entire available energy budget.

### 4.2 Penalty-based cost function

QAOA works by evaluating an objective through a quantum cost operator. In our custom implementation, the constrained optimization problem was represented by a penalty-based cost function:

$$\boxed{C(x)=-\sum_{i=1}^{n}v_i x_i+\lambda\left[\max(0,E(x)-B)\right]^2}$$

where

$$E(x)=\sum_{i=1}^{n}w_i x_i$$

and:

- $C(x)$ is the cost assigned to a candidate selection.
- The negative priority term converts a maximization problem into a minimization problem.
- $E(x)$ is the total energy consumed.
- $B$ is the energy budget.
- $\lambda$ is the penalty coefficient.

In our code, the penalty coefficient was set to:

$$\boxed{\lambda=\max_i(v_i)+1}$$

For the two datasets, this gives $\lambda=10$.

If a selection exceeds the energy budget, the penalty increases its cost. Feasible selections do not receive an energy-violation penalty.

> **Important limitation:** This penalty function discourages infeasible selections but does not mathematically guarantee that every infeasible selection will rank below every feasible selection for every possible dataset. The returned solution must therefore be checked against the energy constraint.

### 4.3 Quantum cost operator

The implementation encoded the candidate-selection costs in a diagonal operator:

$$\boxed{U_C(\gamma)|x\rangle=e^{-i\gamma C(x)}|x\rangle}$$

Here, $\gamma$ is a variational parameter. Each computational-basis state represents a possible combination of selected sites, and the cost operator applies a cost-dependent quantum phase to that state.

In the code, the corresponding operation was constructed using a `DiagonalGate`.

The cost values were scaled numerically before constructing the operator to help with optimization stability. The exact scaling factor should be reported from the notebook if numerical reproducibility is required.

### 4.4 Initial superposition

Each qubit was initialized using a Hadamard gate:

$$\boxed{H|0\rangle=\frac{|0\rangle+|1\rangle}{\sqrt{2}}}$$

For $n$ qubits, this creates an equal superposition over $2^n$ possible binary selections:

$$\boxed{|\psi_0\rangle=\frac{1}{\sqrt{2^n}}\sum_{x\in\{0,1\}^{n}}|x\rangle}$$

Therefore:

- Five qubits represent $2^5=32$ possible selections.
- Nine qubits represent $2^9=512$ possible selections.

This is the mathematical state representation used in the simulation. It does not mean that the system automatically evaluates all candidates and identifies the best answer in one measurement.

### 4.5 Mixer operation

The mixer used single-qubit rotations around the X axis:

$$\boxed{U_M(\beta)=\prod_{i=1}^{n}R_X^{(i)}(2\beta)}$$

where $\beta$ is another variational parameter.

The rotation matrix is:

$$
R_X(\theta)=
\begin{pmatrix}
\cos(\theta/2)&-i\sin(\theta/2)\\
-i\sin(\theta/2)&\cos(\theta/2)
\end{pmatrix}
$$

The mixer changes the amplitudes of the computational-basis states, allowing the quantum state to explore different candidate selections.

### 4.6 QAOA circuit and parameter optimization

The QAOA-inspired circuit applies cost and mixer operations repeatedly:

$$\boxed{|\psi(\boldsymbol{\gamma},\boldsymbol{\beta})\rangle=\prod_{k=1}^{p}U_M(\beta_k)U_C(\gamma_k)|\psi_0\rangle}$$

For this experiment, $p=2$, so the circuit contains two cost layers and two mixer layers.

A classical optimizer, COBYLA, adjusts the variational parameters $\boldsymbol{\gamma}$ and $\boldsymbol{\beta}$ to minimize the expected cost:

$$\boxed{\min_{\boldsymbol{\gamma},\boldsymbol{\beta}}\langle\psi|C|\psi\rangle}$$

This creates a hybrid optimization workflow: a classical optimizer tunes the parameters, while the quantum-state simulation evaluates the resulting objective.

> **Implementation qualification:** The notebook used a custom diagonal-cost, QAOA-inspired circuit rather than the standard QUBO-to-Ising construction and a conventional QAOA implementation. Its mixer also was not constraint-preserving. It is therefore more accurate to describe this as a custom QAOA-inspired simulation than as a fully validated implementation of standard QAOA for constrained knapsack optimization.

### 4.7 Statevector simulation and measurement

The implementation used Qiskit's `Statevector` functionality to represent and evaluate the simulated quantum state.

The state probabilities were then used to sample candidate bitstrings. The experiment used 4,096 shots, meaning up to 4,096 samples were drawn from the simulated measurement distribution.

Each measured bitstring was interpreted as a candidate site selection. The selected sites were evaluated against the priority objective and energy constraint, and the best feasible sampled candidate was retained.

### 4.8 Classical benchmark using exhaustive search

We also used exhaustive classical search to determine the exact optimum for each dataset. Every possible binary selection was evaluated, and the highest-priority feasible selection was identified.

The number of combinations was:

$$N_{\text{combinations}}=2^n$$

Thus, exhaustive search evaluated 32 possible combinations for Processor A and 512 for Processor B.

We compared the sampled QAOA-inspired solution with this classical benchmark. This provided a reference for assessing solution quality, although it did not establish a quantum speed advantage.

## 5. Experimental Results

The following results were obtained from the experiment.

| Metric | Processor A | Processor B |
|---|---|---|
| Qubits | 5 | 9 |
| Candidate sites | 5 | 9 |
| Energy budget | 10 | 10 |
| Selected sites | B, D, E | D, E, F |
| Total priority | 16 | 17 |
| Energy consumed | 10 | 10 |
| Classical optimum | 16 | 17 |
| Optimality gap | 0% | 0% |
| Reported execution time | 1.0163 s | 10.6686 s |

### Interpretation of the results

**Solution quality.** Both configurations found a feasible solution matching the exact classical optimum for their respective datasets. Their observed optimality gap was zero.

**Execution time.** Processor A took approximately 1.02 seconds, compared with 10.67 seconds for Processor B. Processor B's measured runtime was about 10.5 times longer.

**Inspection priority.** Processor B achieved a total priority of 17, compared with 16 for Processor A. However, Processor B had four additional candidate sites, so the two configurations did not solve identical optimization instances.

The optimality gap was calculated as:

Optimality Gap (%) = (V_classical - V_QAOA) / V_classical x 100
For both configurations, the gap was zero because the sampled solution matched the exact optimum for its own dataset.

The reported runtime measures this particular classical simulation and optimization workflow. It is not the execution time of a physical quantum processor, and the comparison alone does not establish that one quantum processor is intrinsically faster than another.

## 6. How Quantum Computing Contributes to the Problem

The potential contribution of quantum computing lies in how it represents and explores combinatorial optimization problems.

1. **Superposition.** Qubits represent combinations of binary decisions in a quantum state. This provides a framework for representing many candidate waypoint selections within a single state.
2. **Cost encoding.** The cost operator encodes the relative desirability of candidate selections through their priorities and energy-violation penalties.
3. **Quantum-state exploration.** Mixer operations alter the quantum state, and the variational parameters control the balance between cost-dependent phase changes and exploration of candidate bitstrings.
4. **Measurement and selection.** Measurement produces candidate bitstrings. These candidates are checked for feasibility and evaluated according to their total priority.

For drone inspection planning, this approach could become useful as the number of candidate locations and the complexity of constraints increase. QAOA is one possible approach to investigating such problems.

However, quantum computing has not yet been shown to provide a performance advantage in this experiment. The circuits were simulated classically, the candidate sets were small, and the classical exhaustive-search benchmark was inexpensive. A genuine quantum advantage would require a controlled comparison on appropriate problem sizes, using well-defined hardware or simulation baselines.

## 7. What Can We Conclude About Processor A?

Based on the observed results, Processor A is the more efficient configuration for the smaller five-site test case.

It achieved the exact classical optimum with five qubits and a reported runtime of approximately one second. Processor B also reached its own classical optimum but required more qubits and approximately 10.5 times as much simulation time.

Nevertheless, the results do not prove that Processor A is universally better. Processor B handled a larger candidate set and achieved a higher total priority, while the two configurations were not tested on identical datasets. A fair comparison would require running both approaches on the same problem instances wherever possible, repeating trials with multiple random seeds, and reporting average runtime and solution quality.

## 8. Final Conclusion — Report-Ready Version

### Conclusion

This study investigated a QAOA-inspired approach to drone waypoint optimization by formulating inspection-site selection as a 0/1 Knapsack Problem. The objective was to maximize the cumulative priority of selected locations while satisfying a fixed energy budget. Two simulated configurations were evaluated: Processor A, comprising five qubits and five candidate sites, and Processor B, comprising nine qubits and nine candidate sites.

The implementation incorporated a penalty-based cost function, Hadamard initialization, a diagonal cost operator, RX mixer operations, classical parameter optimization using COBYLA, and statevector-based sampling. An exhaustive classical search was used to establish the optimal solution for each dataset.

Both configurations achieved solutions matching their respective classical optima. Processor A selected sites B, D, and E, obtaining a total priority of 16 with an energy consumption of 10 units. Processor B selected sites D, E, and F, obtaining a total priority of 17 while also satisfying the energy budget. The reported simulation runtimes were approximately 1.0163 seconds and 10.6686 seconds, respectively.

These results indicate that the five-qubit configuration was more computationally efficient for the smaller test instance, whereas the nine-qubit configuration accommodated a larger set of candidate inspection sites. However, because the datasets differed in size and composition, this experiment does not establish the universal superiority of either configuration. Furthermore, the observed execution times correspond to classical simulation rather than physical quantum hardware.

In conclusion, the experiment demonstrates the feasibility of representing a simplified drone waypoint selection problem using a QAOA-inspired quantum optimization framework. Future work should investigate standard QAOA formulations, more realistic flight-energy constraints, repeated experimental trials, larger benchmark instances, and comparisons against established classical optimization algorithms. These extensions would provide a more rigorous assessment of the potential role of quantum optimization in autonomous drone mission planning.

## 9. Future Improvements

To strengthen the project scientifically and technically, the following extensions are recommended:

1. **Implement standard QAOA:** Construct a well-defined QUBO or Ising cost Hamiltonian from the knapsack problem, with an appropriate treatment of the energy constraint.
2. **Use identical benchmark instances:** Compare five- and nine-qubit configurations on shared candidate sets where feasible, then separately assess scaling with additional sites.
3. **Repeat experiments:** Run multiple random seeds and report mean runtime, variability, feasible-solution rate, and optimality gap.
4. **Compare with classical algorithms:** Evaluate brute-force search, dynamic programming, and integer linear programming.
5. **Model realistic drone constraints:** Incorporate travel distance, battery reserve, flight time, obstacle avoidance, and site dependencies.
6. **Investigate quantum hardware:** If suitable hardware becomes available, assess noise, circuit depth, measurement requirements, and actual hardware execution overhead.
