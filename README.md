# Deep-Learning-from-first-principles-in-Python-R-and-Octave-Part-8

The details of the codeset and plots are included in the attached Microsoft Word Document (.docx) file in this repository. 
You need to view the file in "Read Mode" to see the contents properly after downloading the same.

Deep Learning - A Brief Introduction
=====================================

##1.  Introduction
In this post, I discuss and implement a key functionality needed while building Deep Learning networks viz. 'Gradient Checking'. Gradient Checking is a key method to check the correctness of your implementation, specifically the forward propagation and the backward propagation cycles of an implementation. In addition I also discuss some tips while tuning hyper-parameters.

Gradient Checking  is based on the following. One iteration of Gradient Descent is given by
\theta := \theta \frac{d}{d\theta}J(\theta). To minimize the cost we will need to minimize
J(\theta)

Let g(\theta) be a function that computes the detivative \frac {d}{d\theta}J(\theta). We need to numerically evaluate that the implementation of the function g(\theta) is correct.

The derivative of a function is

    \frac {d}{d\theta}J(\theta) = lim->0 \frac {J(\theta +\epsilon) - J(\theta -\epsilon)} {2*\epsilon}

    *Note:* The above derivative is based on the 2 sided derivative instead of the 1-sided derivative which is given by 

    \frac {d}{d\theta}J(\theta) = lim->0 \frac {J(\theta +\epsilon) - J(\theta)} {\epsilon}

    This is because the error in the 2-sided equation is of O(\epsilon^{2}) as opposed O(\epsilon) for the 1-sided derivative.

    Gradient Check uses the 2 sided derivative as follows.

    g(\theta) = lim->0 \frac {J(\theta +\epsilon) - J(\theta -\epsilon)} {2*\epsilon}

In Gradient Check the following is done

    Run one normal cycle of your implementation
    a) Compute the output activation by running 1 cycle of forward propagation
    b) Compute the cost using the output activation
    c) Compute the gradients using backpropation (grad)

Perform gradient check

    a) Set \theta . Flatten all Weights and biases matrices and vectors to a column vector.
    b) Set \theta+. Bump up theta by adding epsilon (\theta + \epsilon)
    c) Perform forward propagation with \theta+
    d) Compute cost with \theta+ - J(\theta+)
    e) Set \theta-. Bump down theta by subtracting epsilon (\theta - \epsilon)
    f) Perform forward propagation with \theta-
    g) Compute cost with \theta- - J(\theta-)
    h) Compute \frac {d} {d\theta} J(\theta)  or 'gradapprox' as\frac {J(\theta+) - J(\theta-) } {2\epsilon} using the 2 sided derivative.
    i) Compute L2norm or the Euclidean distance between 'grad' and 'gradapprox'. If the
    diference is of the order of 10^{-5} or 10^{-7} the implementation is correct. 

After spending a better part of 3 days I now realize how critical Gradient Check is for ensuring the correctness of you implementation. Initially I was getting very high difference and did not know how to understand the results or debug my implementation. After many hours of staring at the results, I have been able to finally arrive at a way to localize issues in the implementation. In fact, I did catch a small bug in my Python code, which did not exist in the R and Octave implementations. I will demonstrate this below

##1.1a Gradient Check - Sigmoid Activation - Python
```{python}
import numpy as np
import matplotlib

exec(open("DLfunctions8.py").read())
exec(open("testcases.py").read())
train_X, train_Y, test_X, test_Y = load_dataset()
layersDimensions = [2,4,1]  
parameters = initializeDeepModel(layersDimensions)
AL, caches, dropoutMat = forwardPropagationDeep(train_X, parameters, keep_prob=1, hiddenActivationFunc="relu",outputActivationFunc="sigmoid")

cost = computeCost(AL, train_Y, outputActivationFunc="sigmoid") 
print("cost=",cost)

gradients = backwardPropagationDeep(AL, train_Y, caches, dropoutMat, lambd=0, keep_prob=1,                                   hiddenActivationFunc="relu",outputActivationFunc="sigmoid")

epsilon = 1e-7
outputActivationFunc="sigmoid"

# Set-up variables
parameters_values, _ = dictionary_to_vector(parameters)
grad = gradients_to_vector(parameters,gradients)
num_parameters = parameters_values.shape[0]
J_plus = np.zeros((num_parameters, 1))
J_minus = np.zeros((num_parameters, 1))
gradapprox = np.zeros((num_parameters, 1))

# Compute gradapprox using 2 sided derivative
for i in range(num_parameters):
    # Compute J_plus[i]. Inputs: "parameters_values, epsilon". Output = "J_plus[i]".
    thetaplus = np.copy(parameters_values)                                   
    thetaplus[i][0] = thetaplus[i][0] + epsilon                                 
    AL, caches, dropoutMat = forwardPropagationDeep(train_X, vector_to_dictionary(parameters,thetaplus), keep_prob=1, 
                                              hiddenActivationFunc="relu",outputActivationFunc=outputActivationFunc)
    J_plus[i] = computeCost(AL, train_Y, outputActivationFunc=outputActivationFunc) 
    
    
    # Compute J_minus[i]. Inputs: "parameters_values, epsilon". Output = "J_minus[i]".
    thetaminus = np.copy(parameters_values)                                     
    thetaminus[i][0] = thetaminus[i][0] - epsilon                                     
    AL, caches, dropoutMat  = forwardPropagationDeep(train_X, vector_to_dictionary(parameters,thetaminus), keep_prob=1, 
                                              hiddenActivationFunc="relu",outputActivationFunc=outputActivationFunc)     
    J_minus[i] = computeCost(AL, train_Y, outputActivationFunc=outputActivationFunc)                            
   
    
    # Compute gradapprox[i]   
    gradapprox[i] = (J_plus[i] - J_minus[i])/(2*epsilon)

# Compare gradapprox to backward propagation gradients by computing difference. 
numerator = np.linalg.norm(grad-gradapprox)                                           
denominator = np.linalg.norm(grad) +  np.linalg.norm(gradapprox)                                         
difference =  numerator/denominator                                        


if difference > 1e-5:
    print ("\033[93m" + "There is a mistake in the backward propagation! difference = " + str(difference) + "\033[0m")
else:
    print ("\033[92m" + "Your backward propagation works perfectly fine! difference = " + str(difference) + "\033[0m")
print(difference)
print("\n")    
# Covert grad to dictionary
m=vector_to_dictionary2(parameters,grad)
print("Gradients from backprop")
print(m)
print("\n")
# Convert gradapprox to dictionary
n=vector_to_dictionary2(parameters,gradapprox)
print("Gradapprox from gradient check")
print(n)

```


##1.1b Gradient Check - Softmax Activation - Python (Error!!)
In the code below I show, how I managed to spot a bug in your implementation
```{python}
import numpy as np
exec(open("DLfunctions8.py").read())
N = 100 # number of points per class
D = 2 # dimensionality
K = 3 # number of classes
X = np.zeros((N*K,D)) # data matrix (each row = single example)
y = np.zeros(N*K, dtype='uint8') # class labels
for j in range(K):
  ix = range(N*j,N*(j+1))
  r = np.linspace(0.0,1,N) # radius
  t = np.linspace(j*4,(j+1)*4,N) + np.random.randn(N)*0.2 # theta
  X[ix] = np.c_[r*np.sin(t), r*np.cos(t)]
  y[ix] = j


# Plot the data
#plt.scatter(X[:, 0], X[:, 1], c=y, s=40, cmap=plt.cm.Spectral)
layersDimensions = [2,3,3]
y1=y.reshape(-1,1).T
train_X=X.T
train_Y=y1

parameters = initializeDeepModel(layersDimensions)
AL, caches, dropoutMat = forwardPropagationDeep(train_X, parameters, keep_prob=1, 
                                                hiddenActivationFunc="relu",outputActivationFunc="softmax")

cost = computeCost(AL, train_Y, outputActivationFunc="softmax") 
print("cost=",cost)

gradients = backwardPropagationDeep(AL, train_Y, caches, dropoutMat, lambd=0, keep_prob=1, 
                                    hiddenActivationFunc="relu",outputActivationFunc="softmax")
# Note the transpose of the gradients for Softmax has to be taken
L= len(parameters)//2
print(L)
gradients['dW'+str(L)]=gradients['dW'+str(L)].T
gradients['db'+str(L)]=gradients['db'+str(L)].T

gradient_check_n(parameters, gradients, train_X, train_Y, epsilon = 1e-7,outputActivationFunc="softmax")
```

Gradient Check gives a high value of the difference of 0.71996. Inspecting the Gradients and Gradapprox we can see there is a very big  discrepancy in db2. After I went over my code I discovered that I my computation in the function layerActivationBackward for Softmax was 

        
 
   if activationFunc == 'softmax':
        dW = 1/numtraining * np.dot(A_prev,dZ)
        db = np.sum(dZ, axis=0, keepdims=True)
        dA_prev = np.dot(dZ,W)
instead of

   if activationFunc == 'softmax':
        dW = 1/numtraining * np.dot(A_prev,dZ)
        db = **1/numtraining**  np.sum(dZ, axis=0, keepdims=True)
        dA_prev = np.dot(dZ,W)

After fixing this error when I ran Gradient Check I get

##1.1c Gradient Check - Softmax Activation - Python (Corrected!!)
```{python}
import numpy as np
exec(open("DLfunctions8.py").read())
N = 100 # number of points per class
D = 2 # dimensionality
K = 3 # number of classes
X = np.zeros((N*K,D)) # data matrix (each row = single example)
y = np.zeros(N*K, dtype='uint8') # class labels
for j in range(K):
  ix = range(N*j,N*(j+1))
  r = np.linspace(0.0,1,N) # radius
  t = np.linspace(j*4,(j+1)*4,N) + np.random.randn(N)*0.2 # theta
  X[ix] = np.c_[r*np.sin(t), r*np.cos(t)]
  y[ix] = j


# Plot the data
#plt.scatter(X[:, 0], X[:, 1], c=y, s=40, cmap=plt.cm.Spectral)
layersDimensions = [2,3,3]
y1=y.reshape(-1,1).T
train_X=X.T
train_Y=y1

parameters = initializeDeepModel(layersDimensions)
AL, caches, dropoutMat = forwardPropagationDeep(train_X, parameters, keep_prob=1, 
                                                hiddenActivationFunc="relu",outputActivationFunc="softmax")

cost = computeCost(AL, train_Y, outputActivationFunc="softmax") 
print("cost=",cost)

gradients = backwardPropagationDeep(AL, train_Y, caches, dropoutMat, lambd=0, keep_prob=1, 
                                    hiddenActivationFunc="relu",outputActivationFunc="softmax")
# Note the transpose of the gradients for Softmax has to be taken
L= len(parameters)//2
print(L)
gradients['dW'+str(L)]=gradients['dW'+str(L)].T
gradients['db'+str(L)]=gradients['db'+str(L)].T

gradient_check_n(parameters, gradients, train_X, train_Y, epsilon = 1e-7,outputActivationFunc="softmax")
```

        
        
##1.2a Gradient Check - Sigmoid Activation - R 
```{r}
source("DLfunctions8.R")

z <- as.matrix(read.csv("circles.csv",header=FALSE)) 

x <- z[,1:2]
y <- z[,3]
X <- t(x)
Y <- t(y)
#layersDimensions = c(2,4,1) #Works
layersDimensions = c(2,5,1)
parameters = initializeDeepModel(layersDimensions)
retvals = forwardPropagationDeep(X, parameters,keep_prob=1, hiddenActivationFunc="relu",
                                 outputActivationFunc="sigmoid")
AL <- retvals[['AL']]
caches <- retvals[['caches']]
dropoutMat <- retvals[['dropoutMat']]
cost <- computeCost(AL, Y,outputActivationFunc="sigmoid",
                    numClasses=layersDimensions[length(layersDimensions)])
print(cost)

# Backward propagation.
gradients = backwardPropagationDeep(AL, Y, caches, dropoutMat, lambd=0, keep_prob=1, hiddenActivationFunc="relu",
                                    outputActivationFunc="sigmoid",numClasses=layersDimensions[length(layersDimensions)])

epsilon = 1e-07
outputActivationFunc="sigmoid"
parameters_values = list_to_vector(parameters)
grad = gradients_to_vector(parameters,gradients)
num_parameters = dim(parameters_values)[1]
J_plus = matrix(rep(0,num_parameters),
                nrow=num_parameters,ncol=1)
J_minus = matrix(rep(0,num_parameters),
                 nrow=num_parameters,ncol=1)
gradapprox = matrix(rep(0,num_parameters),
                    nrow=num_parameters,ncol=1)

# Compute gradapprox
for(i in 1:num_parameters){
    # Compute J_plus[i]. Inputs: "parameters_values, epsilon". Output = "J_plus[i]".
    
    
    thetaplus = parameters_values                                   
    thetaplus[i][1] = thetaplus[i][1] + epsilon                                 
    retvals = forwardPropagationDeep(X, vector_to_list(parameters,thetaplus), keep_prob=1, 
                                                hiddenActivationFunc="relu",outputActivationFunc=outputActivationFunc)
    
    AL <- retvals[['AL']]
    J_plus[i] = computeCost(AL, Y, outputActivationFunc=outputActivationFunc) 


   # Compute J_minus[i]. Inputs: "parameters_values, epsilon". Output = "J_minus[i]".
    thetaminus = parameters_values                                     
    thetaminus[i][1] = thetaminus[i][1] - epsilon                                     
    retvals  = forwardPropagationDeep(X, vector_to_list(parameters,thetaminus), keep_prob=1, 
                                                 hiddenActivationFunc="relu",outputActivationFunc=outputActivationFunc)     
    AL <- retvals[['AL']]
    J_minus[i] = computeCost(AL, Y, outputActivationFunc=outputActivationFunc)                            


    # Compute gradapprox[i]   
    gradapprox[i] = (J_plus[i] - J_minus[i])/(2*epsilon)
}
# Compare gradapprox to backward propagation gradients by computing difference.


numerator = L2NormVec(grad-gradapprox)                                           
denominator = L2NormVec(grad) +  L2NormVec(gradapprox)                                         
difference =  numerator/denominator 
if(difference > 1e-5){
    cat("There is a mistake, the difference is too high",difference)
} else{
    cat("The implementations works perfectly", difference)
}


# This can be used to check
print("Gradients from backprop")
vector_to_list2(parameters,grad)
print("Grad approx from gradient check")
vector_to_list2(parameters,gradapprox)
```

##1.2b Gradient Check - Softmax Activation - R 
```{r}
source("DLfunctions8.R")
Z <- as.matrix(read.csv("spiral.csv",header=FALSE)) 

# Setup the data
X <- Z[,1:2]
y <- Z[,3]
X <- t(X)
Y <- t(y)
layersDimensions = c(2, 3, 3)

parameters = initializeDeepModel(layersDimensions)
retvals = forwardPropagationDeep(X, parameters,keep_prob=1, hiddenActivationFunc="relu",
                                 outputActivationFunc="softmax")
AL <- retvals[['AL']]
caches <- retvals[['caches']]
dropoutMat <- retvals[['dropoutMat']]
cost <- computeCost(AL, Y,outputActivationFunc="softmax",
                    numClasses=layersDimensions[length(layersDimensions)])
print(cost)

# Backward propagation.
gradients = backwardPropagationDeep(AL, Y, caches, dropoutMat, lambd=0, keep_prob=1, hiddenActivationFunc="relu",
                                    outputActivationFunc="softmax",numClasses=layersDimensions[length(layersDimensions)])
# Need to take transpose of the last layer for Softmax
L=length(parameters)/2
gradients[[paste('dW',L,sep="")]]=t(gradients[[paste('dW',L,sep="")]])
gradients[[paste('db',L,sep="")]]=t(gradients[[paste('db',L,sep="")]])


gradient_check_n(parameters, gradients, X, Y, 
                 epsilon = 1e-7,outputActivationFunc="softmax")
```
##2.1 Tip for tuning hyperparameters

Deep Learning Networks come with a large number of hyper parameters which require tuning. The hyper parameters are

    1. \alpha -learning rate
    2. Number of layers
    3. Number of hidden units
    4. Number of iterations
    5. Momentum - \beta - 0.9
    6. RMSProp - \beta_{1} - 0.9
    7. Adam - \beta_{1},\beta_{2} and \epsilon
    8. learning rate decay
    9. mini batch size
    10. Initialization method - He, Xavier
    11. Regularization

- Among the above the most critical is learning rate decay. Rather than just trying out random values, it may help to try out values on a logarithmic scale. So we could try out
values -0.01,0.1,1.0,10 etc. If we find that the cost is between 0.01 and 0.1 we could use a technique similar to binary search, so we can try 0.01, 0.05. If we need to be bigger than 0.01 and 0.05 we could try 0.25 and etc.

- The performance of Momentum and RMSProp are very good and work well with values 0.9. Even this it is better to try in values of 1-\beta in the logarithimic range. So 1-\beta could 0.001,0.01,0.1 and hence \beta would be 0.999,0.99 or 0.9

- Increasing the number of hidden units or number of hidden layers need to be done gradually. I have noticed that increasing number of hidden layers heavily does not improve performance and sometimes degrades it.

- Sometimes, I tend to increase the number of iterations if I think I see a steady decrease in the cost for a certain learning rate

- It may also help to add learning rate decay if you see there is an oscillation while it decreases.

- Xavier and He initializations also help in a fast convergence and are worth trying out.
