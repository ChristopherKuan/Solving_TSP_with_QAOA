# Solving TSP with QAOA
This is a project about solving The Travelling Salesman Problem (TSP) with Quantum Approximate Optimization Algorithm (QAOA) using Qiskit library and Python.

**Objective of this project**
1. Formulate TSP into QUBO and Ising Hamiltonian in different encoding methods
2. Code to solve TSP with QAOA using Qiskit library
3. Perform sensitivity analysis on different parameters of this project

**Encodings explored**
1. Time slot based encoding<br>
   * The decision variable encodes the node i that is at time slot t
2. Edge based encoding<br>
   * The decision variable encodes the edge ij that is used

The **Documents folder** contains the following files:
1. Time slot encoding QUBO and Ising Formulation
   - Explains on how to formulate TSP as QUBO and convert to Ising Hamiltonian formulation based on time slot encoding
2. Edge based encoding QUBO and Ising Formulation
   - Explains on how to formulate TSP as QUBO and convert to Ising Hamiltonian formulation based on edge based encoding
3. Code explained
   - Explains how to run Qiskit's QAOA
4. Results and discussion
   - Sensitivity analysis of different parameters used in this project, including P layers, amount of nodes, and others.
5. Sources
   - Shows the sources used to make this project possible
  
The **Code folder** contains the following files:
1. TSP_qaoa.ipynb
   - The code on solving TSP with QAOA
2. TSP_qaoa_comparison.ipynb
   - The code on comparing different parameters on this project
3. requirements.txt
   - All the dependencies used for this project
