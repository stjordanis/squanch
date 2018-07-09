.. _teleportationDemo:

Quantum Teleportation
=====================

Quantum teleportation allows two parties that share an entangled pair to transfer a quantum state using classical communication. This process has tremendous applicability to quantum networks, which will need to transfer fragile quantum states between distant nodes. Conecptually, quantum teleportation is the inverse of :ref:`superdense coding <superdenseCodingDemo>`.

The source code for this demo is included in the `demos` directory of the SQUANCH repository.

Protocol
--------

.. image:: https://www.media.mit.edu/quanta/qasm2circ/test2.png

Below is a simple two-party quantum teleportation protocol. We'll be using the above circuit diagram.

	1. Alice generates an EPR pair; for this protocol, we'll use the state :math:`\lvert q_1 q_2 \rangle = \frac{1}{\sqrt{2}} \left (\lvert 00 \rangle + \lvert 11 \rangle \right )`. She will keep one particle in the pair and send the other one to Bob.

	2. Alice entangles her qubit :math:`q_0` with her ancilla :math:`q_1` by applying controlled-not and Hadamard operators. 

	3. Alice measures both of her qubits and communicates the results (two bits) to Bob through a classical channel. Bob's qubit is now in one of four possible states, one of which is :math:`\lvert q_0 \rangle`. Bob will use Alice's two bits to determine what operations to apply to recover :math:`\lvert q_0 \rangle`.

	4. Bob applies a Pauli-X operator to his qubit if Alice's ancilla collapsed to :math:`\lvert q_1 \rangle = \lvert 1 \rangle`, and he applies a Pauli-Z operator to his qubit if her qubit collapsed to :math:`\lvert q_0 \rangle = \lvert 1 \rangle`. 


Implementation
--------------

Quantum teleportation is a simple protocol to implement in any quantum computing simulation framework, but SQUANCH's :ref:`Agent <agent>` and :ref:`Channel <channels>` modules provide an intuitive way to work with sending and receiving qubits, and the :ref:`QStream <qstream>` module allows you to create performant simulations of teleporting a large number of states in succession. 

First, let's import what we'll need.

.. code:: python

	import numpy as np
	import matplotlib.pyplot as plt
	from squanch import *

Now, we'll want to define the behavior of Alice and Bob. We'll extend the :ref:`Agent <agent>` class to create two child classes, and then we can change the `run()` method for each of them. For Alice, we'll want to include logic for creating an EPR pair and sending it to Bob, as well as the subsequent entanglement and measurement logic.

.. code:: python 

	class Alice(Agent):
		'''Alice sends qubits to Bob using a shared Bell pair'''

		def teleport(self, qsystem):
			# Generate a Bell pair and send half of it to Bob
			q, a, b = qsystem.qubits
			H(a)
			CNOT(a, b)
			self.qsend(bob, b)
			# Perform the teleportation
			CNOT(q, a)
			H(q)
			bobZ = q.measure() # If Bob should apply Z
			bobX = a.measure() # If Bob should apply X
			self.csend(bob, [bobX, bobZ])

		def run(self):
			for qsys in self.stream:
				self.teleport(qsys)

Note that you can add arbitrary methods, such as `teleport()`, to agent child classes; just be careful not to overwrite any existing methods other than `run()`, which should always be overwritten. 

For Bob, we'll want to include the logic to receive the pair half from Alice and act on it according to Alice's measurement results.

.. code:: python

	class Bob(Agent):
		'''Bob receives qubits from Alice and measures the results'''

		def run(self):
			measurement_results = []
			for _ in self.stream:
				b = self.qrecv(alice)
				doX, doZ = self.crecv(alice)
				if doX and b is not None: X(b)
				if doZ and b is not None: Z(b)
				measurement_results.append(b.measure())
			self.output(measurement_results)

This logic will allow Alice and Bob to act on a common quantum stream to teleport states to each other. Now we want to actually instantiate a quantum stream and manipulate the initial state of the first qubit (the one to be teleported) in each system of the stream so that we're not just teleporting the :math:`\lvert 0 \rangle` state over and over.

.. code:: python

	# Allocate memory and output structures
	mem = Agent.shared_hilbert_space(3, 10)
	out = Agent.shared_output()

	# Prepare the initial states
	stream = QStream.from_array(mem)
	statesList = [1, 0, 1, 0, 1, 0, 1, 0, 1, 0]
	for state, qsys in zip(statesList, stream):
		q = qsys.qubit(0)
		if state == 1: X(q)  # Flip the qubits corresponding to 1's

For agents to communicate with each other, they must be connected via quantum or classical channels. The `Agent.qconnect` and `Agent.cconnect` methods add a bidirectional quantum or classical channel, repsectively, to two agent instances and take a channel model and kwargs as optional arguments. In this example, we won't worry about a channel model and will just use the default QChannel and CChannel options. Let's create instances for Alice and Bob and connect them appropriately

.. code:: python

	# Make the agents
	alice = Alice(mem)
	bob = Bob(mem, out = out)

	# Connect the agents
	alice.cconnect(bob)
	alice.qconnect(bob)


Finally, let's create Alice and Bob instances, plug in the Hilbert space and output structures, and run the program. Explicitly allocating and passing memory to agents is necessary because each agent spawns and runs in a separate process, which (generally) have separate memory pools. You'll also need to call `agent.start()` for each agent to signal the process to start running, and `agent.join()` to wait for all agents to finish before proceeding in the program.

.. code:: python

	# Run everything
	alice.start(); bob.start()
	alice.join(); bob.join()

	print "Teleported states {} \n" \
		  "Received states   {}".format(statesList, out["Bob"])

Running what we have so far produces the following output:

.. parsed-literal:: 

	Teleported states [1, 0, 1, 0, 1, 0, 1, 0, 1, 0] 
	Received states   [1, 0, 1, 0, 1, 0, 1, 0, 1, 0]

So at least for the simple cases, our implementation seems to be working! Let's do a little more complex test case now. 

We'll now try teleporting an ensemble of identical states :math:`R_{X}(\theta) \lvert 0 \rangle` for several values of :math:`\theta`. We'll then measure each teleported state and see how it compares with the expected outcome.

.. code:: python

	angles = np.linspace(0, 2 * np.pi, 30)  # RX angles to apply
	numTrials = 250  # number of trials for each angle

	# Allocate memory and output structures
	mem = Agent.shared_hilbert_space(3, len(angles) * numTrials)
	out = Agent.shared_output()

	# Prepare the initial states in the stream
	stream = QStream.from_array(mem)
	for angle in angles:
		for _ in range(numTrials):
			q = stream.head().qubit(0)
			RX(q, angle)
	stream.index = 0  # reset the head counter

	# Make the agents
	alice = Alice(mem)
	bob = Bob(mem, out = out)

	# Connect the agents
	alice.connect(bob)

	# Run everything
	alice.start(); bob.start()
	alice.join(); bob.join()

	results = np.array(out["Bob"]).reshape((len(angles), numTrials))
	mean_results = np.mean(results, axis = 1)
	expected_results = np.sin(angles / 2) ** 2
	plt.plot(angles, mean_results, label = 'Observed')
	plt.plot(angles, expected_results, label = 'Expected')
	plt.legend()
	plt.xlabel("$\Theta$ in $R_X(\Theta)$ applied to qubits")
	plt.ylabel("Fractional $\left | 1 \\right >$ population")
	plt.show()

This gives us the following pretty plot.

.. image:: ../img/teleportationRotation.png 

Source code
-----------

The full source code for this demonstration is available in the demos directory of the SQUANCH repository.