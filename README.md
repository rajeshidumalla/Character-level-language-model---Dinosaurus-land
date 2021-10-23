# Character level language model - Dinosaurus land

Welcome to Dinosaurus Island! 65 million years ago, dinosaurs existed, and in this project they are back. Let's assume that you are in charge of a special task. Leading biology researchers are creating new breeds of dinosaurs and bringing them to life on earth, and your job is to give names to these dinosaurs. If a dinosaur does not like its name, it might go beserk, so choose wisely! 

<table>
<td>
<img src="images/dino.jpg" style="width:250;height:300px;">

</td>

</table>

There is collected a list of all the dinosaur names that I could find, and compiled them into this [dataset](dinos.txt). (Feel free to take a look by clicking the previous link.) To create new dinosaur names, I am going to build a character level language model to generate new names. The algorithm will learn the different name patterns, and randomly generate new names. Hopefully this project will keep you and your team safe from the dinosaurs' wrath! 

Major tasks of this project are:

- How to store text data for processing using an RNN 
- How to synthesize data, by sampling predictions at each time step and passing it to the next RNN-cell unit
- How to build a character-level text generation recurrent neural network
- Why clipping the gradients is important

I will begin by loading in some functions that I have provided for you in `rnn_utils`. Specifically, I have access to functions such as `rnn_forward` and `rnn_backward` which are equivalent to those I've implemented in the previous projects. 


```python
import numpy as np
from utils import *
import random
from random import shuffle
```

## 1 - Problem Statement

### 1.1 - Dataset and Preprocessing

Run the following cell to read the dataset of dinosaur names, create a list of unique characters (such as a-z), and compute the dataset and vocabulary size. 


```python
data = open('dinos.txt', 'r').read()
data= data.lower()
chars = list(set(data))
data_size, vocab_size = len(data), len(chars)
print('There are %d total characters and %d unique characters in your data.' % (data_size, vocab_size))
```

    There are 19909 total characters and 27 unique characters in your data.


The characters are a-z (26 characters) plus the "\n" (or newline character), which in this project plays a role similar to the `<EOS>` (or "End of sentence") token. Here it indicates the end of the dinosaur name rather than the end of a sentence.

In the cell below, I did create a python dictionary (i.e., a hash table) to map each character to an index from 0-26. I also created a second python dictionary that maps each index back to the corresponding character character. This will help to figure out what index corresponds to what character in the probability distribution output of the softmax layer. Below, `char_to_ix` and `ix_to_char` are the python dictionaries. 


```python
char_to_ix = { ch:i for i,ch in enumerate(sorted(chars)) }
ix_to_char = { i:ch for i,ch in enumerate(sorted(chars)) }
print(ix_to_char)
```

    {0: '\n', 1: 'a', 2: 'b', 3: 'c', 4: 'd', 5: 'e', 6: 'f', 7: 'g', 8: 'h', 9: 'i', 10: 'j', 11: 'k', 12: 'l', 13: 'm', 14: 'n', 15: 'o', 16: 'p', 17: 'q', 18: 'r', 19: 's', 20: 't', 21: 'u', 22: 'v', 23: 'w', 24: 'x', 25: 'y', 26: 'z'}


### 1.2 - Overview of the model

The model will have the following structure: 

- Initialize parameters 
- Run the optimization loop
    - Forward propagation to compute the loss function
    - Backward propagation to compute the gradients with respect to the loss function
    - Clip the gradients to avoid exploding gradients
    - Using the gradients, update the parameter with the gradient descent update rule.
- Return the learned parameters 
    
<img src="images/rnn.png" style="width:450;height:300px;">
<caption><center> **Figure 1**: Recurrent Neural Network.  </center></caption>

<img width="626" alt="Screen Shot 2021-10-24 at 12 47 12 am" src="https://user-images.githubusercontent.com/56792400/138559213-cd97aae2-f84f-41eb-bbd8-02d3ea195dae.png">

 

## 2 - Building blocks of the model

In this part, I will build two important blocks of the overall model:
- Gradient clipping: to avoid exploding gradients
- Sampling: a technique used to generate characters

Then I will apply these two functions to build the model.

### 2.1 - Clipping the gradients in the optimization loop

In this section I will implement the `clip` function that I will call inside of my optimization loop. Recall that my overall loop structure usually consists of a forward pass, a cost computation, a backward pass, and a parameter update. Before updating the parameters, I will perform gradient clipping when needed to make sure that the gradients are not "exploding," meaning taking on overly large values. 

In the exercise below, I will implement a function `clip` that takes in a dictionary of gradients and returns a clipped version of gradients if needed. There are different ways to clip gradients; I will use a simple element-wise clipping procedure, in which every element of the gradient vector is clipped to lie between some range [-N, N]. More generally, I will provide a `maxValue` (say 10). In this example, if any component of the gradient vector is greater than 10, it would be set to 10; and if any component of the gradient vector is less than -10, it would be set to -10. If it is between -10 and 10, it is left alone. 

<img src="images/clip.png" style="width:400;height:150px;">
<caption><center> **Figure 2**: Visualization of gradient descent with and without gradient clipping, in a case where the network is running into slight "exploding gradient" problems. </center></caption>

Implementing the function below to return the clipped gradients of my dictionary `gradients`. Function takes in a maximum threshold and returns the clipped versions of the gradients. You can check out this [hint](https://docs.scipy.org/doc/numpy-1.13.0/reference/generated/numpy.clip.html) for examples of how to clip in numpy. so I am going touse the argument `out = ...`.


```python
### GRADED FUNCTION: clip

def clip(gradients, maxValue):
    '''
    Clips the gradients' values between minimum and maximum.
    
    Arguments:
    gradients -- a dictionary containing the gradients "dWaa", "dWax", "dWya", "db", "dby"
    maxValue -- everything above this number is set to this number, and everything less than -maxValue is set to -maxValue
    
    Returns: 
    gradients -- a dictionary with the clipped gradients.
    '''
    
    dWaa, dWax, dWya, db, dby = gradients['dWaa'], gradients['dWax'], gradients['dWya'], gradients['db'], gradients['dby']
   
    
    # clip to mitigate exploding gradients, loop over [dWax, dWaa, dWya, db, dby]. (≈2 lines)
    for gradient in [dWax, dWaa, dWya, db, dby]:
        np.clip(gradient,-maxValue , maxValue, out=gradient)
    
    
    gradients = {"dWaa": dWaa, "dWax": dWax, "dWya": dWya, "db": db, "dby": dby}
    
    return gradients
```


```python
np.random.seed(3)
dWax = np.random.randn(5,3)*10
dWaa = np.random.randn(5,5)*10
dWya = np.random.randn(2,5)*10
db = np.random.randn(5,1)*10
dby = np.random.randn(2,1)*10
gradients = {"dWax": dWax, "dWaa": dWaa, "dWya": dWya, "db": db, "dby": dby}
gradients = clip(gradients, 10)
print("gradients[\"dWaa\"][1][2] =", gradients["dWaa"][1][2])
print("gradients[\"dWax\"][3][1] =", gradients["dWax"][3][1])
print("gradients[\"dWya\"][1][2] =", gradients["dWya"][1][2])
print("gradients[\"db\"][4] =", gradients["db"][4])
print("gradients[\"dby\"][1] =", gradients["dby"][1])
```

    gradients["dWaa"][1][2] = 10.0
    gradients["dWax"][3][1] = -10.0
    gradients["dWya"][1][2] = 0.29713815361
    gradients["db"][4] = [ 10.]
    gradients["dby"][1] = [ 8.45833407]


### 2.2 - Sampling

Now assume that the model is trained. I would like to generate new text (characters). The process of generation is explained in the picture below:

<img src="images/dinos3.png" style="width:500;height:300px;">
<caption><center> **Figure 3**: In this picture, we assume the model is already trained. We pass in x(1) = 0 at the first time step, and have the network then sample one character at a time. </center></caption>

<img width="638" alt="Screen Shot 2021-10-24 at 12 50 34 am" src="https://user-images.githubusercontent.com/56792400/138559339-b9660bf9-8f4d-4328-b188-1e295f0f2488.png">

Here is an example of how to use `np.random.choice()`:
```python
np.random.seed(0)
p = np.array([0.1, 0.0, 0.7, 0.2])
index = np.random.choice([0, 1, 2, 3], p = p.ravel())
```

<img width="595" alt="Screen Shot 2021-10-24 at 12 54 46 am" src="https://user-images.githubusercontent.com/56792400/138559420-95744f54-a154-4fa3-b874-4537a431eae8.png">

```python
# GRADED FUNCTION: sample

def sample(parameters, char_to_ix, seed):
    """
    Sample a sequence of characters according to a sequence of probability distributions output of the RNN

    Arguments:
    parameters -- python dictionary containing the parameters Waa, Wax, Wya, by, and b. 
    char_to_ix -- python dictionary mapping each character to an index.
    seed -- used for grading purposes. Do not worry about it.

    Returns:
    indices -- a list of length n containing the indices of the sampled characters.
    """
    
    # Retrieve parameters and relevant shapes from "parameters" dictionary
    Waa, Wax, Wya, by, b = parameters['Waa'], parameters['Wax'], parameters['Wya'], parameters['by'], parameters['b']
    vocab_size = by.shape[0]
    n_a = Waa.shape[1]
    
    
    # Step 1: Create the one-hot vector x for the first character (initializing the sequence generation). (≈1 line)
    x = np.zeros((vocab_size,1))
    # Step 1': Initialize a_prev as zeros (≈1 line)
    a_prev = np.zeros((n_a,1))
    
    # Create an empty list of indices, this is the list which will contain the list of indices of the characters to generate (≈1 line)
    indices = []
    
    # Idx is a flag to detect a newline character, we initialize it to -1
    idx = -1 
    
    # Loop over time-steps t. At each time-step, sample a character from a probability distribution and append 
    # its index to "indices". We'll stop if we reach 50 characters (which should be very unlikely with a well 
    # trained model), which helps debugging and prevents entering an infinite loop. 
    counter = 0
    newline_character = char_to_ix['\n']
    
    while (idx != newline_character and counter != 50):
        
        # Step 2: Forward propagate x using the equations (1), (2) and (3)
        a = np.tanh(np.dot(Wax,x)+np.dot(Waa,a_prev)+b)
        z = np.dot(Wya,a)+by
        y = softmax(z)
        
        # for grading purposes
        np.random.seed(counter+seed) 
        
        # Step 3: Sample the index of a character within the vocabulary from the probability distribution y
        idx = np.random.choice(range(len(y)),p=y.ravel())


        # Append the index to "indices"
        indices.append(idx)
        
        # Step 4: Overwrite the input character as the one corresponding to the sampled index.
        x = np.zeros((vocab_size,1))
        x[idx] = 1
        
        # Update "a_prev" to be "a"
        a_prev = a
        
        # for grading purposes
        seed += 1
        counter +=1
        
    

    if (counter == 50):
        indices.append(char_to_ix['\n'])
    
    return indices
```


```python
np.random.seed(2)
n, n_a = 20, 100
a0 = np.random.randn(n_a, 1)
i0 = 1 # first character is ix_to_char[i0]
Wax, Waa, Wya = np.random.randn(n_a, vocab_size), np.random.randn(n_a, n_a), np.random.randn(vocab_size, n_a)
b, by = np.random.randn(n_a, 1), np.random.randn(vocab_size, 1)
parameters = {"Wax": Wax, "Waa": Waa, "Wya": Wya, "b": b, "by": by}


indices = sample(parameters, char_to_ix, 0)
print("Sampling:")
print("list of sampled indices:", indices)
print("list of sampled characters:", [ix_to_char[i] for i in indices])
```

    Sampling:
    list of sampled indices: [18, 2, 26, 0]
    list of sampled characters: ['r', 'b', 'z', '\n']


## 3 - Building the language model 

It is time to build the character-level language model for text generation. 


### 3.1 - Gradient descent 

In this section I will implement a function performing one step of stochastic gradient descent (with clipped gradients). I will go through the training examples one at a time, so the optimization algorithm will be stochastic gradient descent. As a reminder, here are the steps of a common optimization loop for an RNN:

- Forward propagate through the RNN to compute the loss
- Backward propagate through time to compute the gradients of the loss with respect to the parameters
- Clip the gradients if necessary 
- Update your parameters using gradient descent 

I am going to Implement this optimization process (one step of stochastic gradient descent). 

I've provided the following functions: 

```python
def rnn_forward(X, Y, a_prev, parameters):
    """ Performs the forward propagation through the RNN and computes the cross-entropy loss.
    It returns the loss' value as well as a "cache" storing values to be used in the backpropagation."""
    ....
    return loss, cache
    
def rnn_backward(X, Y, parameters, cache):
    """ Performs the backward propagation through time to compute the gradients of the loss with respect
    to the parameters. It returns also all the hidden states."""
    ...
    return gradients, a

def update_parameters(parameters, gradients, learning_rate):
    """ Updates parameters using the Gradient Descent Update Rule."""
    ...
    return parameters
```


```python
# GRADED FUNCTION: optimize

def optimize(X, Y, a_prev, parameters, learning_rate = 0.01):
    """
    Execute one step of the optimization to train the model.
    
    Arguments:
    X -- list of integers, where each integer is a number that maps to a character in the vocabulary.
    Y -- list of integers, exactly the same as X but shifted one index to the left.
    a_prev -- previous hidden state.
    parameters -- python dictionary containing:
                        Wax -- Weight matrix multiplying the input, numpy array of shape (n_a, n_x)
                        Waa -- Weight matrix multiplying the hidden state, numpy array of shape (n_a, n_a)
                        Wya -- Weight matrix relating the hidden-state to the output, numpy array of shape (n_y, n_a)
                        b --  Bias, numpy array of shape (n_a, 1)
                        by -- Bias relating the hidden-state to the output, numpy array of shape (n_y, 1)
    learning_rate -- learning rate for the model.
    
    Returns:
    loss -- value of the loss function (cross-entropy)
    gradients -- python dictionary containing:
                        dWax -- Gradients of input-to-hidden weights, of shape (n_a, n_x)
                        dWaa -- Gradients of hidden-to-hidden weights, of shape (n_a, n_a)
                        dWya -- Gradients of hidden-to-output weights, of shape (n_y, n_a)
                        db -- Gradients of bias vector, of shape (n_a, 1)
                        dby -- Gradients of output bias vector, of shape (n_y, 1)
    a[len(X)-1] -- the last hidden state, of shape (n_a, 1)
    """
    
    
    
    # Forward propagate through time (≈1 line)
    loss, cache = rnn_forward(X,Y,a_prev,parameters)
    
    # Backpropagate through time (≈1 line)
    gradients, a = rnn_backward(X,Y,parameters,cache)
    
    # Clip your gradients between -5 (min) and 5 (max) (≈1 line)
    gradients = clip(gradients,5)
    
    # Update parameters (≈1 line)
    parameters = update_parameters(parameters,gradients,learning_rate)
    
    
    
    return loss, gradients, a[len(X)-1]
```


```python
np.random.seed(1)
vocab_size, n_a = 27, 100
a_prev = np.random.randn(n_a, 1)
Wax, Waa, Wya = np.random.randn(n_a, vocab_size), np.random.randn(n_a, n_a), np.random.randn(vocab_size, n_a)
b, by = np.random.randn(n_a, 1), np.random.randn(vocab_size, 1)
parameters = {"Wax": Wax, "Waa": Waa, "Wya": Wya, "b": b, "by": by}
X = [12,3,5,11,22,3]
Y = [4,14,11,22,25, 26]

loss, gradients, a_last = optimize(X, Y, a_prev, parameters, learning_rate = 0.01)
print("Loss =", loss)
print("gradients[\"dWaa\"][1][2] =", gradients["dWaa"][1][2])
print("np.argmax(gradients[\"dWax\"]) =", np.argmax(gradients["dWax"]))
print("gradients[\"dWya\"][1][2] =", gradients["dWya"][1][2])
print("gradients[\"db\"][4] =", gradients["db"][4])
print("gradients[\"dby\"][1] =", gradients["dby"][1])
print("a_last[4] =", a_last[4])
```

    Loss = 126.503975722
    gradients["dWaa"][1][2] = 0.194709315347
    np.argmax(gradients["dWax"]) = 93
    gradients["dWya"][1][2] = -0.007773876032
    gradients["db"][4] = [-0.06809825]
    gradients["dby"][1] = [ 0.01538192]
    a_last[4] = [-1.]


### 3.2 - Training the model 

Given the dataset of dinosaur names, I will use each line of the dataset (one name) as one training example. Every 100 steps of stochastic gradient descent, I will sample 10 randomly chosen names to see how the algorithm is doing. Remember to shuffle the dataset, so that stochastic gradient descent visits the examples in random order. 

Follow the instructions and implement `model()`. When `examples[index]` contains one dinosaur name (string), to create an example (X, Y), you can use this:
```python
        index = j % len(examples)
        X = [None] + [char_to_ix[ch] for ch in examples[index]] 
        Y = X[1:] + [char_to_ix["\n"]]
```
Note that we use: `index= j % len(examples)`, where `j = 1....num_iterations`, to make sure that `examples[index]` is always a valid statement (`index` is smaller than `len(examples)`).
The first entry of `X` being `None` will be interpreted by `rnn_forward()` as setting $x^{\langle 0 \rangle} = \vec{0}$. Further, this ensures that `Y` is equal to `X` but shifted one step to the left, and with an additional "\n" appended to signify the end of the dinosaur name. 


```python
# GRADED FUNCTION: model

def model(data, ix_to_char, char_to_ix, num_iterations = 35000, n_a = 50, dino_names = 7, vocab_size = 27):
    """
    Trains the model and generates dinosaur names. 
    
    Arguments:
    data -- text corpus
    ix_to_char -- dictionary that maps the index to a character
    char_to_ix -- dictionary that maps a character to an index
    num_iterations -- number of iterations to train the model for
    n_a -- number of units of the RNN cell
    dino_names -- number of dinosaur names you want to sample at each iteration. 
    vocab_size -- number of unique characters found in the text, size of the vocabulary
    
    Returns:
    parameters -- learned parameters
    """
    
    # Retrieve n_x and n_y from vocab_size
    n_x, n_y = vocab_size, vocab_size
    
    # Initialize parameters
    parameters = initialize_parameters(n_a, n_x, n_y)
    
    # Initialize loss (this is required because we want to smooth our loss, don't worry about it)
    loss = get_initial_loss(vocab_size, dino_names)
    
    # Build list of all dinosaur names (training examples).
    with open("dinos.txt") as f:
        examples = f.readlines()
    examples = [x.lower().strip() for x in examples]
    
    # Shuffle list of all dinosaur names
    shuffle(examples)
    
    # Initialize the hidden state of your LSTM
    a_prev = np.zeros((n_a, 1))
    
    # Optimization loop
    for j in range(num_iterations):
        
        
        
        # Use the hint above to define one training example (X,Y) (≈ 2 lines)
        index = j%len(examples)
        X = [None] + [char_to_ix[ch] for ch in examples[index]]
        Y = X[1:] + [char_to_ix["\n"]]
        
        # Perform one optimization step: Forward-prop -> Backward-prop -> Clip -> Update parameters
        # Choose a learning rate of 0.01
        curr_loss, gradients, a_prev = optimize(X,Y,a_prev,parameters,learning_rate=0.01)  
        
        
        
        # Use a latency trick to keep the loss smooth. It happens here to accelerate the training.
        loss = smooth(loss, curr_loss)

        # Every 2000 Iteration, generate "n" characters thanks to sample() to check if the model is learning properly
        if j % 2000 == 0:
            
            print('Iteration: %d, Loss: %f' % (j, loss) + '\n')
            
            # The number of dinosaur names to print
            seed = 0
            for name in range(dino_names):
                
                # Sample indices and print them
                sampled_indices = sample(parameters, char_to_ix, seed)
                print_sample(sampled_indices, ix_to_char)
                
                seed += 1  # To get the same result for grading purposed, increment the seed by one. 
      
            print('\n')
        
    return parameters
```

By running the following cell, you should observe the model outputting random-looking characters at the first iteration. After a few thousand iterations, the model should learn to generate reasonable-looking names. 


```python
parameters = model(data, ix_to_char, char_to_ix)
```

    Iteration: 0, Loss: 23.093927
    
    Nkzxwtdmfqoeyhsqwasjjjvu
    Kneb
    Kzxwtdmfqoeyhsqwasjjjvu
    Neb
    Zxwtdmfqoeyhsqwasjjjvu
    Eb
    Xwtdmfqoeyhsqwasjjjvu
    
    
    Iteration: 2000, Loss: 27.933209
    
    Livtos
    Hlecagosajaus
    Ixusairhinbweros
    Lda
    Xusairgnnavgosiariihpr
    Ba
    Tos
    
    
    Iteration: 4000, Loss: 25.885133
    
    Ngytosaurus
    Inda
    Ivusedonnomurusebrggiseantexamanbetalhangagodsauru
    Nda
    Xusmaneriaupsauius
    Ca
    Torangosaurus
    
    
    Iteration: 6000, Loss: 24.717962
    
    Ngytosaurus
    Incaadosaurus
    Kwtosaurus
    Ndaadosaurus
    Xustanasaurus
    Caacosaurus
    Tosaurus
    
    
    Iteration: 8000, Loss: 24.247544
    
    Mhytosaurus
    Inga
    Kxtosaurus
    Ng
    Xstololosaurus
    Da
    Tosaurus
    
    
    Iteration: 10000, Loss: 23.657721
    
    Ngytptan
    Kidaalosaurus
    Kutosaurus
    Nacalosaurus
    Xussaurus
    Daakosaurus
    Tosaurus
    
    
    Iteration: 12000, Loss: 23.468681
    
    Nixrosaurus
    Inecamosaurus
    Kurosaurus
    Nbaahosaurus
    Wrosaurus
    Caaerosaurus
    Stoinasaurus
    
    
    Iteration: 14000, Loss: 23.318021
    
    Onxuspandys
    Kngaagstelanorrepitencumatops
    Lytrochesamvhosiador
    Ola
    Xustangosaurus
    Da
    Strameronthrosaurus
    
    
    Iteration: 16000, Loss: 23.135437
    
    Nixuskanesaurus
    Ingbaisaurus
    Kyuusaurus
    Olaaiton
    Wussangosaurus
    Cabaspaechus
    Trodonosaurus
    
    
    Iteration: 18000, Loss: 23.080210
    
    Meutosaurus
    Inga
    Jurorgongnaulops
    Macaeosaurus
    Vusrapendosaurus
    Daafosaurus
    Trodonnonsaurus
    
    
    Iteration: 20000, Loss: 22.875113
    
    Lixutrimariasaurus
    Hrceaatroe
    Hyutolialleximis
    Lecakosaurus
    Vrrodon
    Caacoracensaurus
    Susaurus
    
    
    Iteration: 22000, Loss: 22.738468
    
    Lixosaurus
    Hoga
    Hyxskangoraverataphomplanossaug
    Lecagrus
    Xrus
    Cebcosaurus
    Strbianodun
    
    
    Iteration: 24000, Loss: 22.689156
    
    Ilussaurus
    Eracalosaurus
    Eustrasaurus
    Iabalosaurus
    Vustangosaurus
    Agalosaurus
    Tromiangavenkuchus
    
    
    Iteration: 26000, Loss: 22.553910
    
    Ilvusia
    Fracalosaurus
    Hustranchiauloprasaurus
    Iba
    Ustodia
    Aeacosaurus
    Sungkapigus
    
    
    Iteration: 28000, Loss: 22.684028
    
    Keustrimarpes
    Endaagosaurus
    Gurosaurus
    Kecaerosaurus
    Ustrimanpatesaurus
    Baacosaurus
    Surapenorscoptappelto
    
    
    Iteration: 30000, Loss: 22.513617
    
    Liwusterasaurus
    Hima
    Hywtsaurus
    Ledalosaurus
    Vusthonosaurus
    Adahusaurus
    Trtarasaurus
    
    
    Iteration: 32000, Loss: 22.418336
    
    Levusaurus
    Hibaalosaurus
    Iussaurus
    Lacagspelapus
    Ustrangosaurus
    Ba
    Trodon
    
    
    Iteration: 34000, Loss: 22.506671
    
    Ilusosaurus
    Eracaerur
    Futrodonbmaveshpateosaurus
    Icechusaurus
    Ustrhieosaurus
    Adagropechus
    Surchanoslispoclongspanthus
    
    


## Conclusion

You can see that the algorithm has started to generate plausible dinosaur names towards the end of the training. At first, it was generating random characters, but towards the end you could see dinosaur names with cool endings. Feel free to run the algorithm even longer and play with hyperparameters to see if you can get even better results. My implemetation generated some really cool names like `maconucon`, `marloralus` and `macingsersaurus` and also learned that dinosaur names tend to end in `saurus`, `don`, `aura`, `tor`, etc.

If the model generates some non-cool names, don't blame the model entirely--not all actual dinosaur names sound cool. (For example, `dromaeosauroides` is an actual dinosaur name and is in the training set.) But this model should give you a set of candidates from which you can pick the coolest! 

This project had used a relatively small dataset, so that I could train an RNN quickly on a CPU. Training a model of the english language requires a much bigger dataset, and usually needs much more computation, and could run for many hours on GPUs. I ran the dinosaur name for quite some time, and so far our favoriate name is the great, undefeatable, and fierce: Mangosaurus!

<img src="images/mangosaurus.jpeg" style="width:250;height:300px;">

## 4 - Writing like Shakespeare

Since it's quite fun and informative. 

A similar (but more complicated) task is to generate Shakespeare poems. Instead of learning from a dataset of Dinosaur names I am using a collection of Shakespearian poems. Using LSTM cells, I can learn longer term dependencies that span many characters in the text--e.g., where a character appearing somewhere a sequence can influence what should be a different character much later in this sequence. These long term dependencies were less important with dinosaur names, since the names were quite short. 


<img src="images/shakespeare.jpg" style="width:500;height:400px;">
<caption><center> Let's become poets! </center></caption>

I have implemented a Shakespeare poem generator with Keras below. Run the following cell to load the required packages and models. This may take a few minutes. 


```python
from __future__ import print_function
from keras.callbacks import LambdaCallback
from keras.models import Model, load_model, Sequential
from keras.layers import Dense, Activation, Dropout, Input, Masking
from keras.layers import LSTM
from keras.utils.data_utils import get_file
from keras.preprocessing.sequence import pad_sequences
from shakespeare_utils import *
import sys
import io
```

    Using TensorFlow backend.


    Loading text data...
    Creating training set...
    number of training examples: 31412
    Vectorizing training set...
    Loading model...


    C:\Users\being\AppData\Local\conda\conda\envs\tf\lib\site-packages\keras\models.py:291: UserWarning: Error in loading the saved optimizer state. As a result, your model is starting with a freshly initialized optimizer.
      warnings.warn('Error in loading the saved optimizer '


To save some time, I have already trained a model for ~1000 epochs on a collection of Shakespearian poems called [*"The Sonnets"*](shakespeare.txt). 

Let's train the model for one more epoch. When it finishes training for an epoch---this will also take a few minutes---you can run `generate_output`, which will prompt asking you for an input (`<`40 characters). The poem will start with your sentence, and our RNN-Shakespeare will complete the rest of the poem for you! For example, try "Forsooth this maketh no sense " (don't enter the quotation marks). Depending on whether you include the space at the end, your results might also differ--try it both ways, and try other inputs as well. 



```python
print_callback = LambdaCallback(on_epoch_end=on_epoch_end)

model.fit(x, y, batch_size=128, epochs=1, callbacks=[print_callback])
```

    Epoch 1/1
    31412/31412 [==============================] - 40s 1ms/step - loss: 2.7373





    <keras.callbacks.History at 0x19d2b765f60>




```python
# Run this cell to try with different inputs without having to re-train the model 
generate_output()
```

    Write the beginning of your poem, the Shakespeare machine will complete it. Your input is: the Shakespeare machine will complete it
    
    
    Here is your poem: 
    
    the Shakespeare machine will complete it shight
    labpure my mide fiarl the may bidn which dide,
    even but sicarn is na felfeon by eyes in thy,
    hid wats nor labker this i bely beside,
    lide that lide the shate but thou gordted,
    that your love mas sake i makce feam.
    for showl fase in thus bemfot to eving in thy creaptery trat,
    noataous being wallming to gra save their swarts,
    and with dither wady every broing one,
    and i her can yes wo rerteo

The RNN-Shakespeare model is very similar to the one I have built for dinosaur names. The only major differences are:
- LSTMs instead of the basic RNN to capture longer-range dependencies
- The model is a deeper, stacked LSTM model (2 layer)
- Using Keras instead of python to simplify the code 

If you want to learn more, you can also check out the Keras Team's text generation implementation on GitHub: https://github.com/keras-team/keras/blob/master/examples/lstm_text_generation.py.

Thanks for Reading this notebook! 

**References**:
- This exercise took inspiration from Andrej Karpathy's implementation: https://gist.github.com/karpathy/d4dee566867f8291f086. To learn more about text generation, also check out Karpathy's [blog post](http://karpathy.github.io/2015/05/21/rnn-effectiveness/).
- For the Shakespearian poem generator, our implementation was based on the implementation of an LSTM text generator by the Keras team: https://github.com/keras-team/keras/blob/master/examples/lstm_text_generation.py 
