---
layout: single
title: T9 Hacks Handwriting Neural Net Post Mortem 
---

I&#39;ve been busy really lately so I haven&#39;t had a lot of time to work on any of my projects or write any posts. However, a couple weeks ago I did get a chance to participate in the T9Hacks Hackathon sponsored by Major League Hacks. It was a really great experience and I wanted to write a post about the project my team worked on.

I&#39;m also trying out a new way to integrate code snippets which makes it a bit easier for me to add them – we&#39;ll see how it goes.

### The Project

During the hackathon, we sort of lacked focus during the hackathon. We started out working on a web project with a Java backend. However, as we started making progress, we realized that none of us were actually enjoying working on the project. So, we decided to completely switch gears a couple hours into the hackathon. Since our time was pretty limited we decided to focus on learning something new. That way we wouldn&#39;t be too worried about the success of our final project.

So, we decided to work on a handwriting recognition neural net. Our goal was to achieve a decent level of accuracy reading the MNIST dataset. This wasn&#39;t really the most original thing we could do, but none of us had any experience working on neural nets and we thought it would be a good place to start. We also thought it would be pretty cool if we could get it working.

Of course, we needed to have a name for our project. We settled on Num (as Aaron, one of my teammates, liked to call it, The Numerical Understanding Machine) since it read numbers. [Our Devpost](https://devpost.com/software/num) does a good job explaining the project.

### Version 1 – Hackathon Version – [Repository](https://github.com/Spaceman1701/T9HackProject)

During the hackathon I was responsible for the actual neural net component of the project, while the rest of my team was responsible for the image processing, I/O, and marketing (because we thought it would be fun to win the marketing award). In the end, the only thing that didn&#39;t work was my code (oops). We did get surprisingly close to finishing the project.

Before I get into what went wrong, I&#39;ll go over what the project was. We used Python 3.6 and NumPy (though I didn&#39;t end up using NumPy for anything). The network is simple enough in theory: one Python file with a Network class.  The user can then create an instance of the class with the desired number of layers and their sizes. The class has a train function and a feed\_forward function. The train function takes the training data (in an awful format – but that&#39;s just a product of the time constraints) and trains the network using gradient decent backpropagation. The feed\_forward function takes input data and outputs the result of running it through the network.

```python
net = Network([784, 30, 10])

net.train(data, 30, 10, 8)

print(net.feed_forward(test_input))
```

*Example of the network in use*

While I originally wanted to approach the problem functionally, I didn&#39;t end up doing that. The Network class (contained in network\_large.py) contains a list of lists containing nodes. Each inner list represents a layer. Every node in a layer must be connected to every Node in the previous layer. Each node is represented by a Node class. Each connection between Nodes is represented by an Edge class. Each edge has a field for its weight. It&#39;s probably apparent by now that this approach is fairly overcomplicated. Debugging was made difficult because the Network had no direct way to access its weights. Architecturally, it forced flow control for backpropagation to be handled by each the Node class. From a conceptual standpoint, this doesn&#39;t make a lot of sense because propagation happens layer by layer.

It becomes a lot more clear how problematic this is when looking at the code for error calculation:

```python
def calc_error(self, expected):
	if self.error and not self.needs_error:
		return self.error
	self.error = 0
	self.bias_error = 0
	if not self.outs:
		self.error = (expected - self.value)
	else:
		self.error += sum([e.weight * e.target.calc_error(expected) for e in self.outs])
		self.bias_error += self.bias * self.cost_derivative(expected)
	self.needs_error = False
	return self.error
```
*Node.calc\_error(). Note: During the hackathon this did include bias offset calculation. It was removed in a failed attempt to fix a problem*

There are probably a couple things that should jump out right away with this function. Firstly, the fact that there is only one argument is a bit odd. Since the function calculates the error from the node&#39;s calculated value and the expected value, it would make sense for the function to take two parameters, calculated (or activation) and expected. The reason is doesn&#39;t is because the node actually keeps the results of its last activation as a class field. The calc\_error function assumes that the eval function has already been called.

Next you&#39;ll notice the if statement in the first line. That&#39;s a result of each Node storing its own calculated error. Since each node ought only to calculate its error once per backpropagation, the node has to remember whether it has calculated its error already. At the end of backpropagation, every node has needs\_error set to True.

Then you&#39;ll notice the messy logic that actually calculates the error…. And you&#39;re probably starting to get the point.

The function almost has a recursive design to it in that it calls calc\_error internally. It&#39;s not truly recursive though, as it calls calc\_error on other nodes. It does keep all the debugability and readability issues of recursive functions, however.

At this point it ought to be clear that this version of the neural net was a mess. This type of decentralized and side-effect prone code can be found all over the project.  Unfortunately at the hackathon I made an error in my code for calculation the error (I still don&#39;t know where) and we were never able to get any sort of statistically significant results. This was, of course, everyone on the teams&#39; first attempt at writing at neural net, so we didn&#39;t expect to have great results.

We did end up winning the marketing award though, so it wasn&#39;t a total failure…

### Version 2 – The One that Actually Works – [Repository](https://github.com/Spaceman1701/T9HackUpdated)

After coming so close to finishing during the hackathon, I really wanted to actually finish the neural net. So, I did, sorta. I&#39;m still having some trouble with backpropagation on networks that have multiple hidden layers. Despite that issue, I was able to achieve almost 90% recognition of the MNIST data. I&#39;m pretty happy with that.

In order to achieve this, I had to completely rewrite the neural net with a more functional approach to the architecture. Because the decentralized nature of the hackathon version made it so difficult to debug, I wanted to make sure that the code was clear and that functionality wasn&#39;t so spread out. I also wanted to avoid writing code with tons of hard to track side effects.

So, after doing some research, [I found a website](http://neuralnetworksanddeeplearning.com/chap1.html)which gave me the critical realization that all of the operations needed to do backpropagation could be represented as matrix operations. I decided to write my own matrix class instead of using NumPy pretty much just for the fun of it.

The Network class works pretty much exactly the same from the outside as it did before. However, internally it uses completely vectoredized operations. Just look at the new backpropagation function (the error calculation is just a few lines now, not its own function like in the hackathon version):

```python
def backpropagate(self, inputs, outputs):
	weights_offset = [Matrix(next_layer, current_layer).set_zero()
					for next_layer, current_layer in zip(self.sizes[1:], self.sizes[:-1])]
	biases_offset = [Matrix(size, 1).set_zero() for size in self.sizes[1:]]

	inputs_mat = Matrix.from_list(inputs)
	outputs_mat = Matrix.from_list(outputs)

	activation_transfers = [inputs_mat]
	d_activations = [calc_d_sigmoid(inputs_mat)]

	for layer in range(self.num_layers - 1):
		activation = self.calc_activation(layer, activation_transfers[-1])
		activation_transfers.append(calc_sigmoid(activation))
		d_activations.append(calc_d_sigmoid(activation_transfers[-1]))

	error = calc_error(activation_transfers[-1], outputs_mat).entrywise_product(d_activations[-1])
	weights_offset[-1] = error * activation_transfers[-2].transpose()
	biases_offset[-1] = error

	for layer in range(self.num_layers - 2, 1, -1):
		d_activation = d_activations[layer]
		error = (self.weights[layer + 1].transpose() * error).entrywise_product(d_activation)
		weights_offset[layer] = error * activation_transfers[layer - 1].transpose()
		biases_offset[layer] = error
	return weights_offset, biases_offset
```

This is obviously much more clear.  The first half of the function just initializes the necessary lists, does a simple forward propagation (the first for loop) while keeping track of some useful intermediary data and calculates the error for the output layer. The actual backpropagation algorithm is just one for loop with 4 lines of code in it at the end of the function (unfortunately, there&#39;s still  a bug in that piece of code, the network works best when it has only an input and output layer which is exactly when that code does not run).

This version was vastly easier to debug because of the changes I made, and I am really happy with the results. It&#39;s probably clear that I was heavily influence by the website I linked previously, but it is also still very much my own code.

While there is still a lot of work that I could do, I&#39;m not really planning on working on this anymore. I&#39;m happy with the success rate of the network as it is, and I&#39;m excited to get back to work on some of my other projects. I do plan on doing more work with neural nets in the future, however. I found this project super interesting and I think the results are really cool – it can actually read handwriting!