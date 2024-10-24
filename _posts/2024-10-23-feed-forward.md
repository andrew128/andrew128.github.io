---
layout: post
title: Multi Layer Perceptrons (with code)
---

Transformers feed the output of self attention blocks into a feed forward layer. 
We will look at one such example, a Multilayer Perceptron (MLP).

## Post Outline
- [What is an MLP](#what-is-an-mlp)
- [Quick Recap of a Perceptrony](#quick-recap-of-a-perceptron)
- [MLP is an extension of a perceptron](#mlp-is-an-extension-of-a-perceptron)
- [NumPy Implementation](#numpy-implementation)
- [NumPy Training](#numpy-training)
- [PyTorch Implementation](#pytorch-implementation)
- [Further Exploration](#further-exploration)
- [Resources](#resources)

## What is an MLP
A feed forward neural network is one where information flows in one direction from inputs to outputs (i.e. no loops or cycles in the network like RNNs).
An MLP is a specific type of feed forward network consisting of an input layer, one or more hidden layers, and an output layer. 

## Quick Recap of a Perceptron
We previously covered [single layer perceptrons](https://andrew128.github.io/Perceptron/) which can only solve linearly separable problems

The simplest perceptron is a binary classifier of input data.
The goal is to learn a function that maps input x (real value vector) to a single binary value.
The binary function consists of a weight vector and a bias vector.

A prediction is simply performing a dot product between the weight matrix and the input vector, adding the bias vector, and then putting it through a step activation function.
The output is a binary 1 or 0. 

To learn the values in the weight matrix and bias vector, we iterate over a predetermined number of training epochs. 
We perform a prediction and compare it to the actual label for the training input.
The error is calculated by subtracting the label from the prediction. 
Then the matrix is updated by multiplying each value in the input matrix by the learning rate and the error.

The [Perceptron convergence theorem](https://en.wikipedia.org/wiki/Perceptron#Convergence_of_one_perceptron_on_a_linearly_separable_dataset) states that for a linearly separable dataset, the binary perceptron classifier will converge after a certain number of mistakes.
That number is a function of R and gamma. 
R, the max euclidean distance from any point to the origin, represents the radius of the smallest hypersphere centered at the origin that encompasses all the data points. 
Gamma represents the margin of separation between the decision boundary and the closest data point to that decision boundary. 
For larger R, it will take longer to converge.
For larger gamma, it will be faster to converge.

The binary perceptron predicts one of two output classes but there are ways to extend to more classes. 
[Here is an example repo](https://github.com/siddk/multiclass_perceptron) using the Multi-class decision rule.
From the readme, "the data comes in the same way" but the input data is multiplied by a separate weight vector for each class. 
The dot product that yields the highest value is the class the data belongs to. 
During training, when a correct classification is made, nothing happens.
When an incorrect classification is made, the input feature vector is subtracted from the weight vector of the incorrectly predicted class and added to the feature of the correctly predicted class. 
This means that in the future, the value for the correctly predicted class on the same input class will be larger than it was before the current iteration (i.e. a step in the right direction).

## MLP is an extension of a perceptron
An MLP consists of one or more hidden layers, an input layer, and an output layer. 
An MLP is an extension of the perceptron.
Whereas the perceptron only had a single weight vector to multiple, the MLP consists of hidden layers where each hidden layer represents the output of another weight vector calculation. 
Each neuron in a layer is connected to all of the neurons in the subsequent layer with its own weight.
All the weights are represented in a matrix. 
There are multiple weight matrixes, one between each pair of adjacent layers. 
Each hidden layer's output is the result of a matrix multiplication followed by an activation function.

The activation function is something that each neuron applies to the weighted sum of its inputs. 
The activation functions are non linear, allowing the neural network to learn complex patterns in data. 
Without non linear activation functions the MLP could only learn linear relationships (regardless of how many layers it has). 

## Numpy Implementation
To concretize the previous section, lets walk through an MLP implementation from scratch (using numpy). 

Here is the initialization of the MLP. 
We have a single hidden layer where $$W_1$$ and $$b_1$$ connect the input and the hidden layer and $$W_2$$ and $$b_2$$ connect the hidden and the output layer. 

```
def __init__(self, input_size, hidden_size, output_size):
    self.input_size = input_size
    self.hidden_size = hidden_size
    self.output_size = output_size
    
    # Initialize weights and biases
    self.W1 = np.random.randn(input_size, hidden_size) / np.sqrt(input_size)
    self.b1 = np.zeros((1, hidden_size))
    self.W2 = np.random.randn(hidden_size, output_size) / np.sqrt(hidden_size)
    self.b2 = np.zeros((1, output_size))
```
The weight matrix values are initialized randomly according to a normal distribution, which is commonly used for networks using ReLU activation functions. 

The bias values are initialization to zero.
This is common practice because biases don't have the same issues that can occur with zero-initialized weights. 

Here is what the forward pass looks like. It involves a matrix multiplication, then a relu function, then the second matrix multiplication, then the softmax layer. 

```
def relu(self, x):
    return np.maximum(0, x)

def softmax(self, x):
    exp_x = np.exp(x - np.max(x, axis=1, keepdims=True))
    return exp_x / np.sum(exp_x, axis=1, keepdims=True)

def forward(self, X):
    self.z1 = np.dot(X, self.W1) + self.b1
    self.a1 = self.relu(self.z1)
    self.z2 = np.dot(self.a1, self.W2) + self.b2
    self.a2 = self.softmax(self.z2)
    return self.a2
```

The input X has the dimensions "number of samples by number of input features". 
The first layer weights W1 has the dimensions "number of input features by number of hidden layer neurons". 
The first layer bias b1 has the dimensions "1 by number of hidden layer neurons"
After performing the W1/b1 operations on the input matrix (i.e. z1), we get a matrix of size "number of samples in dataset by number of hidden layer neurons".
This is then shoved through the relu function which is applied elementwise to each value in the matrix but doesn't change the overall dimensions. 

The same operations are performed with the W2/b2, only now the input dimension is the number of hidden layer neurons and the output dimension is the number of output classes. 
Because the input is still "number of samples in dataset by number of hidden layer neurons", that is how we retain the number of samples dimension. 
The output has the dimensions "number of samples by the number of output classes". 
Then the softmax function is applied rowwise to each sample where each row is 1 by the number of output classes. 
The softmax function subtracts the max value in the row from each element for numerical stability (helping to prevent numerical overflow for the exponent computation). 
The exponent function $$e^x$$ is then applied to each element in the row, ensuring that all values are positive. 
Then the values are normalized by summing up all of the exponent values and dividing each exponent value by the sum. 

The final output is "the number of samples in the dataset by the number of output classes". 
This can be interpreted as the probability distribution over all the output classes for each input. 

## Numpy training
Over a predetermined number of epochs, the model weights are trained.
First the model performs inference on the input.
Then the loss is calculated using cross entropy loss. 
Then the `backward()` function is called which updates the weights with the learning rate hyperparameter. 

```
def train(self, X, y, epochs, learning_rate):
    self.learning_rate = learning_rate
    for epoch in range(epochs):
        output = self.forward(X)
        loss = self.cross_entropy_loss(y, output)
        self.backward(X, y, output)
        
        if epoch % 100 == 0:
            print(f"Epoch {epoch}, Loss: {loss}")
```

We already covered the `forward()` function previously. 
We then calculate the loss using `cross_entropy_loss`.
Cross entropy is a measure of the difference between two probability distributions. 
In this case, the probability distributions that we're interested in are the predictions and the one hot encoded label vector. 

The cross entropy loss is used in the function `backward()` which performs gradient descent. 
The purpose of gradient descent is to minimize the loss function (cross entropy in this case) for a set of parameters and a given set of inputs. 
The idea is that by the end of a predetermine number of epochs, the weights and biases are adjusted to better predict the training labels for a set of inputs. 

We do this by stepping in the direction of steepest descent as defined by the negative of the gradient.
The step size is controlled by the learning rate. 

In order to calculate the gradients in a neural network, we need to perform backpropagation. 
Backpropagation propagates the error backwards through the network to adjust each layer's weights and biases. 
It does this by calculating the gradient of the loss function w.r.t each parameter in the network and applying it. 

```
def backward(self, X, y, output):
    batch_size = X.shape[0]
    
    # Gradient of the loss with respect to the output
    self.output_delta = output - y  # For cross-entropy loss and softmax output
    
    # Gradient for the hidden-to-output weights
    self.dW2 = (1 / batch_size) * np.dot(self.a1.T, self.output_delta)
    self.db2 = (1 / batch_size) * np.sum(self.output_delta, axis=0, keepdims=True)
    
    # Gradient for the input-to-hidden weights
    self.z1_delta = np.dot(self.output_delta, self.W2.T) * self.relu_derivative(self.a1)
    self.dW1 = (1 / batch_size) * np.dot(X.T, self.z1_delta)
    self.db1 = (1 / batch_size) * np.sum(self.z1_delta, axis=0, keepdims=True)
    
    # Update weights and biases
    self.W2 -= self.learning_rate * self.dW2
    self.b2 -= self.learning_rate * self.db2
    self.W1 -= self.learning_rate * self.dW1
    self.b1 -= self.learning_rate * self.db1
```

Stochastic gradient descent is a variant of the gradient descent algorithm that is much more efficient. 
The difference between gradient descent and stochastic gradient descent (SGD) is that SGD uses a small batch of the entire dataset to compute the gradient as opposed to the entire dataset. 
This can help escape local minima when descending the gradient by adding extra randomness as well. 

## PyTorch Implementation
[Here's](https://github.com/andrew128/multi-layer-perceptron/blob/main/mlp_pytorch.py) the full code for the pytorch implementation.
Functionally it's the exact same as the numpy code. 
We just call the PyTorch API. 

## Further exploration
- Math
  - Proof of the Perceptron convergence theorem: if the training set is linearly separable then the perceptron is guaranteed to converge after making finitely many mistakes
  - backpropagation math
- Non feed forward neural networks like RNNs
- Non fully connected networks like CNNs
- parallelized training 
- comparison and analysis of various activation functions (what layers they're used, how they influence weight initialization)
- explore various methods of initialization of weights and biases
- compare sizes of neural networks (e.g. adding more hidden layers, increasing the sizes of hidden layers)
- regularization techniques to prevent overfitting
- cross validation to assess model's performance and generalizability
- using GPU/parallel computing techniques to improve performance
- normalization of input data
- one hot encoding of labels
- various loss functions (cross entropy for classification, MSE for regression)

## Resources
- [3Blue1Brown How might LLMs store facts](https://www.youtube.com/watch?v=9-Jl0dxWQs8&ab_channel=3Blue1Brown)
- [3Blue1Brown What is backpropagation really doing](https://www.youtube.com/watch?v=Ilg3gGewQ5U&list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi&index=3&ab_channel=3Blue1Brown)
- [LLM Visualization at bbycroft](https://bbycroft.net/llm)
- [Nielsen book](http://neuralnetworksanddeeplearning.com/chap1.html#perceptrons)
- [MLP Numpy code](https://github.com/andrew128/multi-layer-perceptron/blob/main/mlp_numpy.py)
- [MLP PyTorch code]()