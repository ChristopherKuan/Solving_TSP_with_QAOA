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

The sampler is simply setup with the code <br>
`sampler = StatevectorSampler(seed=state_vector_sampler_seed) `

On the other hand, the optimizer is the classical optimizer that QAOA uses. In this case, COBYLA is used. <br>
`optimizer = COBYLA()`

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

Now, we this representation in mind, we move to constructing the Ising Hamiltonian in code. The code used in this case is Qiskit's `SparsePauliOp.from_sparse_list(pauli_list, num_qubits)`. The `pauli_list` is a list containing tuples of the Pauli Z information, where each tuple contains the gate you want to use (can be X, Z or any gate), a list of gate position(s), and the coefficient. It is constructed by the function `build_paulis(cost_matrix, penalty, n)`. This function takes in the cost matrix (distance matrix), the penalty coefficient, and number of nodes as argument.

Within the `build_paulis()` function, we first construct the linear terms, which are:

$$
H = - \frac{1}{2}\sum_{i,j,t} d_{ij}Z_{it} + A \left(2-n \right)\sum_{it}Z_{it}
$$

This is done by the code:

```
for i in range(row):
  for j in range(col):
    weight = sum(cost_matrix[i][:]) * -0.5 + penalty*(2-n)
    pauli_list.append(("Z", [i*n+j], weight))
```

We have to loop through all possible combinations of nodes and times, thus a nested for loop is used to traverse across the 2D cost matrix. Next, since the TSP is a fully connected graph, the weight is computed by simply summing each row of the matrix and multiplying by -0.5, corresponding to the first linear term. As for the second linear term, it is simply penalty mulitplied by 2-n, same as the formula. The pauli list is appended with the tuple containing one 'Z' as it is a linear term, with position calculated with the formula we was while collapsing into 1D matrix, and the coefficient is the calculated weight.

Next, we build the quadratic terms which are

$$
H = \frac{1}{4}\sum_{i,j,t}d_{ij}Z_{it}Z_{jt+1} + \frac{A}{2}\sum_i\sum_{t \lt t'} Z_{it}Z_{i't} + \frac{A}{2}\sum_t\sum_{i \lt i'} Z_{it}Z_{i't}
$$

with the code

```
for i in range(row*col):
  for j in range(i+1, row*col):
    from_node = int(math.floor(i / n))
    from_time = int(i % n)
    to_node = int(math.floor(j / n))
    to_time = int(j % n)
    weight = 0
    weight = cost_matrix[from_node][to_node] * 0.25 * int(from_time != to_time) + penalty / 2 * int(from_node == to_node) + penalty / 2 * int(from_time == to_time)
    pauli_list.append(("ZZ", [i, j], weight))
```

Similarly, we use a nested loop to go through all the combinations, but this time it is the combinations of 1D matrices. This means we are computing Z<sub>0</sub>Z<sub>1</sub>, Z<sub>0</sub>Z<sub>2</sub>, ..., Z<sub>n<sup>2</sup>-2</sub>Z<sub>n<sup>2</sup>-1</sub>. Within each loop, we first convert back the 1D representation into 2D, to get the to/from nodes and times. This is done with the formula

$$
i = \left\lfloor \frac{k}{n} \right\rfloor
$$

$$
t = k \text{ mod } n
$$

which corresponds to the first 4 lines within the nested loop. Next, the coefficient is calculated by simply converting the equation to code. A difference between the quadratic and linear term is the feasibility checker. `int(from_time != to_time)` ensures that only valid time combinations are computed for the objective function. `int(from_time == to_time)` ensures that only invalud time combinations are appended the penalty value.

The pauli list for quadratic share the same structure as linear terms, except that now there are two Zs like "ZZ", and the position is a list `[i, j]`.

After constructing the pauli list, the function `SparsePauliOp.from_sparse_list(pauli_list, num_qubits=num_qubits)` is called to construct the Ising Hamiltonian, and then normalized with the helper function `normalize_hamiltonian`.

### Constructing QAOA
The QAOA ansatz is constructed with the function `QAOAAnsatz(cost_operator, reps)`. The `cost_operator` takes in the normalized hamiltonian as argument, while the `reps` decides on how many layers of QAOA to produce. Next, the `measure_all()` function is called to add measurement to the end qubits after the QAOA.

### Setting up training (Adapted from QISKIT's QAOA tutorial)
In this code, an AER simulator is used. It can be set up with the following code

```
backend = AerSimulator()
pm = generate_preset_pass_manager(optimization_level=3, backend=backend)
candidate_circuit = pm.run(circuit)
```

where `circuit` is the QAOA ansatz that was constructed previously.

After that, the initial parameters are declared by

```
init_params = [initial_gamma] * p_layer_advanced + [initial_beta] * p_layer_advanced
```

The number of beta and gamma elements in the list must each match the number of QAOA layers

Following that, we define the objective function for the QAOA to minimize:

```
def cost_func_estimator(params, ansatz, hamiltonian, estimator):
    isa_hamiltonian = hamiltonian.apply_layout(ansatz.layout)    
    pub = (ansatz, isa_hamiltonian, params)

    job = estimator.run([pub])
    
    results = job.result()[0]
    cost = results.data.evs
    
    objective_func_vals.append(cost)
    
    return cost
```

Its job is to:
<ol>
  <li>Run the quantum circuit with some parameters (beta, gamma)</li>
  <li>Compute the expected energy (cost)</li>
  <li>Return the value to the optimizer</li>
</ol>

The `isa_hamiltonian` represents the Hamiltonian that has been matched to the physical qubit mapping after transpilation. This is because during transpilation, logical qubits may be rearranged, so the Hamiltonian must match this new ordering. Without this, the observable could be measured on the wrong qubits, giving incorrect energies.

The `pub` bundles everything the estimator needs. The `cost` is the expectation value of the Hamiltonian, which is the result after running the circuit.

### Training QAOA (Adapted from QISKIT's QAOA tutorial)
The training of QAOA then occurs in the following code section:

```
objective_func_vals = [] # Global variable

advanced_start_time = time.time()
# Training
with Session(backend=backend) as session:
    estimator = Estimator(mode=session)
    estimator.options.default_shots = 1000
    
    # Set simple error suppression/mitigation options
    estimator.options.dynamical_decoupling.enable = True
    estimator.options.dynamical_decoupling.sequence_type = "XY4"
    estimator.options.twirling.enable_gates = True
    estimator.options.twirling.num_randomizations = "auto"
    
    # Scipy minimize routine, minimize function does Minimization of scalar 
    # function of one or more variables.
    result = minimize(
        cost_func_estimator, # Objective function to be minimized
        init_params, # initial guess
        args=(candidate_circuit, cost_hamiltonian, estimator), # extra arguments passed to the objective function and its derivatives
        method=method, # type of solver
        tol=tol, # tolerance for termination
    )
    
# Final loop to get result
optimized_circuit = candidate_circuit.assign_parameters(result.x)

sampler = Sampler(mode=backend)
sampler.options.default_shots = 10000

sampler.options.dynamical_decoupling.enable = True
sampler.options.dynamical_decoupling.sequence_type = "XY4"
sampler.options.twirling.enable_gates = True
sampler.options.twirling.num_randomizations = "auto"

pub = (optimized_circuit,)
job = sampler.run([pub], shots=int(1e4))
counts_int = job.result()[0].data.meas.get_int_counts()
counts_bin = job.result()[0].data.meas.get_counts()
shots = sum(counts_int.values())
final_distribution_int = {key: val / shots for key, val in counts_int.items()}
final_distribution_bin = {key: val / shots for key, val in counts_bin.items()}
advanced_end_time = time.time()
advanced_runtime = advanced_end_time - advanced_start_time
```

In the first part (the `with Session` part), the QAOA iteratively finds the lowest energy ground state of the Hamiltonian. The Scipy minimize routine is used to minimize the `cost_func_estimator()`, along with the initial parameters set previously. The classical part used in this case is COBYQA, as QAOA landscapes are highly nonlinear and COBYQA models local curvature using quadratic approximations. COBYQA often converges faster and to better solutions with fewer circuit evaluations. A tolerance for termination is also set to ensure better results.

After searching for the ground state, in the second part, the loop is run one final time to get the final result.

### Result (Advanced)
The results shown in the code are:
<ul>
  <li>Runtime (in seconds)</li>
  <li>Loss values over iterations</li>
  <li>Top 10 most likely bits along with their probabilities</li>
  <li>Optimal path achieved</li>
</ul>

## 4.0 Helper functions
There are 6 helper functions used in this code:
1. `draw_graph`
  - input: graph, color_list, position_list
  - output: none (only displays the rendering of the graph)
2. `normalize_hamiltonian`
  - input: SparsePauliOp's hamiltonian
  - output: SparsePauliOp's normalized hamiltonian
3. `check_feasibility`
  - input: bitstring ('00110101')
  - output: 0 if valid, 1 if invalid
4. `filter_result`
  - input: dictionary of result, where keys are bitstring, values are the probability
  - output: sorted and filtered dictionary of result
5. `plot_result_distribution`
  - input: filtered dictionary of result, top x number of results
  - output: no output, only plots bar graph
6. `show_path`
  - input: filtered dictionary of result
  - output: no output, only print optimal path
