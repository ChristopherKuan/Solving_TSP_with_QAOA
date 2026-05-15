# Code explanation
## Introduction
In this section, we explore how the python code works. There are three main sections of the code:
<ol>
  <li>Getting the graph</li>
  <li>Simple QAOA (using Qiskit built-in functions)</li>
  <li>Advanced QAOA (constructing your own Hamiltonian and QAOA)</li>
</ol>
There are helper functions to assist with different parts of the code, which we will go through in the end.

## 1.0 Getting the graph
First, let us look at the arguments used for getting the graph

```
n = 3 (int)
node_color = "r" (str)
```

"n" is the number of nodes for your TSP. It has to be a positive integer as there cannot be negative node or a percentage of a node.

"node_color" is the color you wish to have your node as when displaying it. You can put in the following colors:
<ul>
  <li>"r" - red</li>
  <li>"g" - green</li>
  <li>"b" - blue</li>
  <li>"y" - yellow</li>
  <li>"m" - magenta</li>
</ul>

Refer to [here](https://networkx.org/documentation/stable/reference/generated/networkx.drawing.nx_pylab.draw_networkx.html) for more colors.

```
num_qubits = n**2
tsp = Tsp.create_random_instance(n, seed=123)
cost_matrix = nx.to_numpy_array(tsp.graph)
print("distance\n", cost_matrix)

colors = [node_color for node in tsp.graph.nodes]
pos = [tsp.graph.nodes[node]['pos'] for node in tsp.graph.nodes]
draw_graph(tsp.graph, colors, pos)
```

In the code, the number of qubits is calculated as the square of the number of nodes. Then a TSP instance is created with the 'TSP' object from `qiskit_optimization.applications` library. The cost matrix is shown then the graph is drawn.

## 2.0 Simple QAOA
### Converting to QUBO then Ising Hamiltonian
In simple QAOA, the QUBO and Ising Hamiltonian is obtained with the following code:

```
qp = tsp.to_quadratic_program()

penalty_weight = 280
qp2qubo = QuadraticProgramToQubo(penalty_weight)
qubo = qp2qubo.convert(qp)

simple_op, offset = qubo.to_ising()

normalized_simple_op = normalize_hamiltonian(simple_op)
```

The first part uses the `to_quadratic_program` of the `tsp` instance to convert the TSP problem into its objective functions and constraints. 

Next, the functions and constraints are converted into QUBO with the `convert()` function of the `QuadraticProgramToQubo` class. Note that the penalty weight is set during the class initialization. The penalty weight can be decided with the following formula:

$$
\text{Penalty weight} = \text{max coefficient of QUBO} \times \text{number of qubits}
$$

In the third part, QUBO is converted to Ising Hamiltonian simply with the code `to_ising() `. 

Finally, the Hamiltonian is normalized with the helper function `normalize_hamiltonian()` to help facilitate training of QAOA.

### Defining sampler and optimizer
A sampler's job is to run the quantum circuit, measure probabilities or bitstrings, and return the sampling results. It assumes:
<ul>
  <li>Exact Statevector simulation</li>
  <li>Noiseless</li>
  <li>No real hardware execution</li>
</ul>

The sampler is simply setup with the code `sampler = StatevectorSampler(seed=state_vector_sampler_seed) `

On the other hand, the optimizer is the classical optimizer that QAOA uses. In this case, COBYLA is used. `optimizer = COBYLA()`

### Training QAOA (simple)
The QAOA is initiated with the code `qaoa = QAOA(sampler=sampler, optimizer=optimizer, reps=p_layer_simple, callback=callback)`, where `p_layer_simple` decides how deep the QAOA is and `callback` is the function to help record the loss values and parameters throughout the training.

### Result (simple)
The results shown in the code are:
<ul>
  <li>Runtime (in seconds)</li>
  <li>Loss values over iterations</li>
  <li>Top 10 most likely bits along with their probabilities</li>
  <li>Optimal path achieved</li>
</ul>

## 3.0 Advanced QAOA
### Constructing Ising Hamiltonian
Decision variables are originally defined as

$$
x_{it} =
\begin{cases}
1 & \text{if the salesman visits node } i \text{ at time } t \\
0 & \text{otherwise}
\end{cases}
$$

However, we need the decision variables to correspond to qubits indexed by a single label. So we have reduce the dimension of decision variables from 2D into 1D. This can be achieved with the following substitution:

$$
x_{ij} \rightarrow x_{in+t} \rightarrow x_k, k \in \left\\{0,...,n^2-1\right\\}
$$

Now the decision variables are in 1D and it corresponds to the Pauli Z matrices. Let there be 3 nodes (n = 3), if we look at decision variable of salesman being at node 2 and time 1, it corresponds to the following qubit representation (where I is the identity matrix):

$$
x_{21} \rightarrow x_7 \rightarrow Z_7 \rightarrow I_8Z_7I_6I_5I_4I_3I_2I_1I_0
$$

The one above shows an example of linear terms in the Ising Hamiltonian. As for quadratic terms, it is as follow:

$$
x_{00}x_{21} \rightarrow x_0x_7 \rightarrow Z_0Z_7 \rightarrow I_8Z_7I_6I_5I_4I_3I_2I_1Z_0
$$
