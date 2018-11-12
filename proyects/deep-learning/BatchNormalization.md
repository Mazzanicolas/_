
# Batch Normalization
One way to make deep networks easier to train is to use more sophisticated optimization procedures such as SGD+momentum, RMSProp, or Adam. Another strategy is to change the architecture of the network to make it easier to train. One idea along these lines is batch normalization which was recently proposed by [3].

The idea is relatively straightforward. Machine learning methods tend to work better when their input data consists of uncorrelated features with zero mean and unit variance. When training a neural network, we can preprocess the data before feeding it to the network to explicitly decorrelate its features; this will ensure that the first layer of the network sees data that follows a nice distribution. However even if we preprocess the input data, the activations at deeper layers of the network will likely no longer be decorrelated and will no longer have zero mean or unit variance since they are output from earlier layers in the network. Even worse, during the training process the distribution of features at each layer of the network will shift as the weights of each layer are updated.

The authors of [3] hypothesize that the shifting distribution of features inside deep neural networks may make training deep networks more difficult. To overcome this problem, [3] proposes to insert batch normalization layers into the network. At training time, a batch normalization layer uses a minibatch of data to estimate the mean and standard deviation of each feature. These estimated means and standard deviations are then used to center and normalize the features of the minibatch. A running average of these means and standard deviations is kept during training, and at test time these running averages are used to center and normalize features.

It is possible that this normalization strategy could reduce the representational power of the network, since it may sometimes be optimal for certain layers to have features that are not zero-mean or unit variance. To this end, the batch normalization layer includes learnable shift and scale parameters for each feature dimension.

[3] Sergey Ioffe and Christian Szegedy, "Batch Normalization: Accelerating Deep Network Training by Reducing
Internal Covariate Shift", ICML 2015.


```python
# As usual, a bit of setup
from __future__ import print_function
import time
import numpy as np
import matplotlib.pyplot as plt
from cs231n.classifiers.fc_net import *
from cs231n.data_utils import get_CIFAR10_data
from cs231n.gradient_check import eval_numerical_gradient, eval_numerical_gradient_array
from cs231n.solver import Solver

%matplotlib inline
plt.rcParams['figure.figsize'] = (10.0, 8.0) # set default size of plots
plt.rcParams['image.interpolation'] = 'nearest'
plt.rcParams['image.cmap'] = 'gray'

# for auto-reloading external modules
# see http://stackoverflow.com/questions/1907993/autoreload-of-modules-in-ipython
%load_ext autoreload
%autoreload 2

def rel_error(x, y):
  """ returns relative error """
  return np.max(np.abs(x - y) / (np.maximum(1e-8, np.abs(x) + np.abs(y))))
```


```python
# Load the (preprocessed) CIFAR10 data.

data = get_CIFAR10_data()
for k, v in data.items():
  print('%s: ' % k, v.shape)
```

    y_test:  (1000,)
    X_val:  (1000, 3, 32, 32)
    y_val:  (1000,)
    y_train:  (49000,)
    X_train:  (49000, 3, 32, 32)
    X_test:  (1000, 3, 32, 32)


## Batch normalization: Forward
In the file `cs231n/layers.py`, we have implemented for you the batch normalization forward pass for the **training** stage in the function `batchnorm_forward`. Check the implementation and make sure you completely understand it. Then, run the following to test this implementation.


```python
# Check the training-time forward pass by checking means and variances
# of features both before and after batch normalization

# Simulate the forward pass for a two-layer network
np.random.seed(231)
N, D1, D2, D3 = 200, 50, 60, 3
X = np.random.randn(N, D1)
W1 = np.random.randn(D1, D2)
W2 = np.random.randn(D2, D3)
a = np.maximum(0, X.dot(W1)).dot(W2)

print('Before batch normalization:')
print('  means: ', a.mean(axis=0))
print('  stds: ', a.std(axis=0))

# Means should be close to zero and stds close to one
print('After batch normalization (gamma=1, beta=0)')
a_norm, _ = batchnorm_forward(a, np.ones(D3), np.zeros(D3), {'mode': 'train'})
print('  mean: ', a_norm.mean(axis=0))
print('  std: ', a_norm.std(axis=0))

# Now means should be close to beta and stds close to gamma
gamma = np.asarray([1.0, 2.0, 3.0])
beta = np.asarray([11.0, 12.0, 13.0])
a_norm, _ = batchnorm_forward(a, gamma, beta, {'mode': 'train'})
print('After batch normalization (nontrivial gamma, beta)')
print('  means: ', a_norm.mean(axis=0))
print('  stds: ', a_norm.std(axis=0))
```

    Before batch normalization:
      means:  [ -2.3814598  -13.18038246   1.91780462]
      stds:  [27.18502186 34.21455511 37.68611762]
    After batch normalization (gamma=1, beta=0)
      mean:  [ 5.55111512e-17 -3.83026943e-17  4.85028684e-17]
      std:  [0.99999999 1.         1.        ]
    After batch normalization (nontrivial gamma, beta)
      means:  [11. 12. 13.]
      stds:  [0.99999999 1.99999999 2.99999999]


## Batch normalization: Forward (testing)
In the file `cs231n/layers.py`, implement the batch normalization forward pass for the **test** stage in the function `batchnorm_forward`. Once you have done so, run the following to test your implementation.


```python
# Check the test-time forward pass by running the training-time
# forward pass many times to warm up the running averages, and then
# checking the means and variances of activations after a test-time
# forward pass.
np.random.seed(231)
N, D1, D2, D3 = 200, 50, 60, 3
W1 = np.random.randn(D1, D2)
W2 = np.random.randn(D2, D3)

bn_param = {'mode': 'train'}
gamma = np.ones(D3)
beta = np.zeros(D3)
for t in range(50):
  X = np.random.randn(N, D1)
  a = np.maximum(0, X.dot(W1)).dot(W2)
  batchnorm_forward(a, gamma, beta, bn_param)
bn_param['mode'] = 'test'
X = np.random.randn(N, D1)
a = np.maximum(0, X.dot(W1)).dot(W2)
a_norm, _ = batchnorm_forward(a, gamma, beta, bn_param)

# Means should be close to zero and stds close to one, but will be
# noisier than training-time forward passes.
print('After batch normalization (test-time):')
print('  means: ', a_norm.mean(axis=0))
print('  stds: ', a_norm.std(axis=0))
```

    After batch normalization (test-time):
      means:  [-0.03927354 -0.04349152 -0.10452688]
      stds:  [1.01531428 1.01238373 0.97819988]


# Question:
Why do the training and testing phases of Batch Normalization need to be implemented differently? What is the difference between the two phases?

# Answer:

En training se calcula la media y la varianza bandose en el mini-batch, mientras que en test se utiliza la poblacion y no el minibach.

src: https://arxiv.org/pdf/1502.03167v3.pdf pg.3 - 4

# Question:

During training BN keeps an exponentially decaying running mean of the mean and variance of each feature. Why shouldn't we directly use the average mean in all iterations?

# Answer:

Relizarlo de esta forma hace que el gradiente y la nomalizacion se contraresten

`It is natural to ask whether we could simply use the moving averages µ, σ to perform the normalization during training, since this would remove the dependence of the normalized activations on the other example in the minibatch.
This, however, has been observed to lead to the model blowing up. As argued in, such use of moving averages would cause the gradient optimization and the normalization to counteract each other.`

src: https://arxiv.org/pdf/1702.03275.pdf pg.2

## Batch Normalization: backward
Now check the implementation of the backward pass for batch normalization in the function `batchnorm_backward`.

To derive the backward pass one should write out the computation graph for batch normalization and backprop through each of the intermediate nodes. Some intermediates may have multiple outgoing branches and you have to be sure that gradients are being summed across these branches in the backward pass.

Once you have a complete understanding of the backward pass, run the following to numerically check the backward pass.


```python
# Gradient check batchnorm backward pass
np.random.seed(231)
N, D = 4, 5
x = 5 * np.random.randn(N, D) + 12
gamma = np.random.randn(D)
beta = np.random.randn(D)
dout = np.random.randn(N, D)

bn_param = {'mode': 'train'}
fx = lambda x: batchnorm_forward(x, gamma, beta, bn_param)[0]
fg = lambda a: batchnorm_forward(x, a, beta, bn_param)[0]
fb = lambda b: batchnorm_forward(x, gamma, b, bn_param)[0]

dx_num = eval_numerical_gradient_array(fx, x, dout)
da_num = eval_numerical_gradient_array(fg, gamma.copy(), dout)
db_num = eval_numerical_gradient_array(fb, beta.copy(), dout)

_, cache = batchnorm_forward(x, gamma, beta, bn_param)
dx, dgamma, dbeta = batchnorm_backward(dout, cache)
print('dx error: ', rel_error(dx_num, dx))
print('dgamma error: ', rel_error(da_num, dgamma))
print('dbeta error: ', rel_error(db_num, dbeta))

```

    dx error:  1.6674604875341426e-09
    dgamma error:  7.417225040694815e-13
    dbeta error:  2.379446949959628e-12


## Fully Connected Nets with Batch Normalization
Now that you have a working implementation for batch normalization, go back to your `FullyConnectedNet` in the file `cs2312n/classifiers/fc_net.py`. Modify your implementation to add batch normalization.

Concretely, when the flag `use_batchnorm` is `True` in the constructor, you should insert a batch normalization layer before each ReLU nonlinearity. The outputs from the last layer of the network should not be normalized. Once you are done, run the following to gradient-check your implementation.

HINT: You might find it useful to define an additional helper layer similar to those in the file `cs231n/layer_utils.py`. If you decide to do so, do it in the file `cs231n/classifiers/fc_net.py`.


```python
np.random.seed(231)
N, D, H1, H2, C = 2, 15, 20, 30, 10
X = np.random.randn(N, D)
y = np.random.randint(C, size=(N,))

for reg in [0, 3.14]:
  print('Running check with reg = ', reg)
  model = FullyConnectedNet([H1, H2], input_dim=D, num_classes=C,
                            reg=reg, weight_scale=5e-2, dtype=np.float64,
                            use_batchnorm=True)

  loss, grads = model.loss(X, y)
  print('Initial loss: ', loss)

  for name in sorted(grads):
    f = lambda _: model.loss(X, y)[0]
    grad_num = eval_numerical_gradient(f, model.params[name], verbose=False, h=1e-5)
    print('%s relative error: %.2e' % (name, rel_error(grad_num, grads[name])))
  if reg == 0: print()
```

    Running check with reg =  0
    Initial loss:  2.2611955101340957
    W1 relative error: 1.10e-04
    W2 relative error: 3.35e-06
    W3 relative error: 3.75e-10
    b1 relative error: 2.22e-03
    b2 relative error: 2.22e-08
    b3 relative error: 9.06e-11
    beta1 relative error: 7.33e-09
    beta2 relative error: 1.17e-09
    gamma1 relative error: 7.47e-09
    gamma2 relative error: 3.35e-09
    
    Running check with reg =  3.14
    Initial loss:  6.996533220108303
    W1 relative error: 1.98e-06
    W2 relative error: 2.28e-06
    W3 relative error: 1.11e-08
    b1 relative error: 5.55e-09
    b2 relative error: 2.22e-08
    b3 relative error: 2.23e-10
    beta1 relative error: 6.32e-09
    beta2 relative error: 3.48e-09
    gamma1 relative error: 5.94e-09
    gamma2 relative error: 3.72e-09


# Batchnorm for deep networks
Run the following to train a six-layer network on a subset of 1000 training examples both with and without batch normalization.


```python
np.random.seed(231)
# Try training a very deep net with batchnorm
hidden_dims = [100, 100, 100, 100, 100]

num_train = 1000
small_data = {
  'X_train': data['X_train'][:num_train],
  'y_train': data['y_train'][:num_train],
  'X_val': data['X_val'],
  'y_val': data['y_val'],
}

weight_scale = 2e-2
bn_model = FullyConnectedNet(hidden_dims, weight_scale=weight_scale, use_batchnorm=True)
model = FullyConnectedNet(hidden_dims, weight_scale=weight_scale, use_batchnorm=False)

bn_solver = Solver(bn_model, small_data,
                num_epochs=10, batch_size=50,
                update_rule='adam',
                optim_config={
                  'learning_rate': 1e-3,
                },
                verbose=True, print_every=200)
bn_solver.train()

solver = Solver(model, small_data,
                num_epochs=10, batch_size=50,
                update_rule='adam',
                optim_config={
                  'learning_rate': 1e-3,
                },
                verbose=True, print_every=200)
solver.train()
```

    (Iteration 1 / 200) loss: 2.340974
    (Epoch 0 / 10) train acc: 0.106000; val_acc: 0.111000
    (Epoch 1 / 10) train acc: 0.300000; val_acc: 0.243000
    (Epoch 2 / 10) train acc: 0.432000; val_acc: 0.314000
    (Epoch 3 / 10) train acc: 0.465000; val_acc: 0.293000
    (Epoch 4 / 10) train acc: 0.562000; val_acc: 0.331000
    (Epoch 5 / 10) train acc: 0.564000; val_acc: 0.315000
    (Epoch 6 / 10) train acc: 0.614000; val_acc: 0.345000
    (Epoch 7 / 10) train acc: 0.683000; val_acc: 0.352000
    (Epoch 8 / 10) train acc: 0.712000; val_acc: 0.311000
    (Epoch 9 / 10) train acc: 0.773000; val_acc: 0.319000
    (Epoch 10 / 10) train acc: 0.766000; val_acc: 0.297000
    (Iteration 1 / 200) loss: 2.302332
    (Epoch 0 / 10) train acc: 0.123000; val_acc: 0.130000


    /Users/decemberlabs/Desktop/postgrado/assignment2/cs231n/optim.py:146: RuntimeWarning: invalid value encountered in sqrt
      next_x = x - config['learning_rate'] * mt / (np.sqrt(vt) + config['epsilon'])


    (Epoch 1 / 10) train acc: 0.260000; val_acc: 0.215000
    (Epoch 2 / 10) train acc: 0.319000; val_acc: 0.273000
    (Epoch 3 / 10) train acc: 0.336000; val_acc: 0.280000
    (Epoch 4 / 10) train acc: 0.386000; val_acc: 0.299000
    (Epoch 5 / 10) train acc: 0.455000; val_acc: 0.325000
    (Epoch 6 / 10) train acc: 0.486000; val_acc: 0.324000
    (Epoch 7 / 10) train acc: 0.498000; val_acc: 0.300000
    (Epoch 8 / 10) train acc: 0.577000; val_acc: 0.311000
    (Epoch 9 / 10) train acc: 0.619000; val_acc: 0.332000
    (Epoch 10 / 10) train acc: 0.654000; val_acc: 0.332000


Run the following to visualize the results from two networks trained above. You should find that using batch normalization helps the network to converge much faster.


```python
plt.subplot(3, 1, 1)
plt.title('Training loss')
plt.xlabel('Iteration')

plt.subplot(3, 1, 2)
plt.title('Training accuracy')
plt.xlabel('Epoch')

plt.subplot(3, 1, 3)
plt.title('Validation accuracy')
plt.xlabel('Epoch')

plt.subplot(3, 1, 1)
plt.plot(solver.loss_history, 'o', label='baseline')
plt.plot(bn_solver.loss_history, 'o', label='batchnorm')

plt.subplot(3, 1, 2)
plt.plot(solver.train_acc_history, '-o', label='baseline')
plt.plot(bn_solver.train_acc_history, '-o', label='batchnorm')

plt.subplot(3, 1, 3)
plt.plot(solver.val_acc_history, '-o', label='baseline')
plt.plot(bn_solver.val_acc_history, '-o', label='batchnorm')
  
for i in [1, 2, 3]:
  plt.subplot(3, 1, i)
  plt.legend(loc='upper center', ncol=4)
plt.gcf().set_size_inches(15, 15)
plt.show()
```

    /Users/decemberlabs/anaconda3/lib/python3.5/site-packages/matplotlib/cbook/deprecation.py:107: MatplotlibDeprecationWarning: Adding an axes using the same arguments as a previous axes currently reuses the earlier instance.  In a future version, a new instance will always be created and returned.  Meanwhile, this warning can be suppressed, and the future behavior ensured, by passing a unique label to each axes instance.
      warnings.warn(message, mplDeprecation, stacklevel=1)



![png](./Batchnormalization/output_16_1.png)


# Batch normalization and initialization
We will now run a small experiment to study the interaction of batch normalization and weight initialization.

The first cell will train 8-layer networks both with and without batch normalization using different scales for weight initialization. The second layer will plot training accuracy, validation set accuracy, and training loss as a function of the weight initialization scale.


```python
np.random.seed(231)
# Try training a very deep net with batchnorm
hidden_dims = [50, 50, 50, 50, 50, 50, 50]

num_train = 1000
small_data = {
  'X_train': data['X_train'][:num_train],
  'y_train': data['y_train'][:num_train],
  'X_val': data['X_val'],
  'y_val': data['y_val'],
}

bn_solvers = {}
solvers = {}
weight_scales = np.logspace(-4, 0, num=20)
for i, weight_scale in enumerate(weight_scales):
  print('Running weight scale %d / %d' % (i + 1, len(weight_scales)))
  bn_model = FullyConnectedNet(hidden_dims, weight_scale=weight_scale, use_batchnorm=True)
  model = FullyConnectedNet(hidden_dims, weight_scale=weight_scale, use_batchnorm=False)

  bn_solver = Solver(bn_model, small_data,
                  num_epochs=10, batch_size=50,
                  update_rule='adam',
                  optim_config={
                    'learning_rate': 1e-3,
                  },
                  verbose=False, print_every=200)
  bn_solver.train()
  bn_solvers[weight_scale] = bn_solver

  solver = Solver(model, small_data,
                  num_epochs=10, batch_size=50,
                  update_rule='adam',
                  optim_config={
                    'learning_rate': 1e-3,
                  },
                  verbose=False, print_every=200)
  solver.train()
  solvers[weight_scale] = solver
```

    Running weight scale 1 / 20


    /Users/decemberlabs/Desktop/postgrado/assignment2/cs231n/optim.py:146: RuntimeWarning: invalid value encountered in sqrt
      next_x = x - config['learning_rate'] * mt / (np.sqrt(vt) + config['epsilon'])


    Running weight scale 2 / 20
    Running weight scale 3 / 20
    Running weight scale 4 / 20
    Running weight scale 5 / 20
    Running weight scale 6 / 20
    Running weight scale 7 / 20
    Running weight scale 8 / 20
    Running weight scale 9 / 20
    Running weight scale 10 / 20
    Running weight scale 11 / 20
    Running weight scale 12 / 20
    Running weight scale 13 / 20
    Running weight scale 14 / 20
    Running weight scale 15 / 20
    Running weight scale 16 / 20
    Running weight scale 17 / 20
    Running weight scale 18 / 20
    Running weight scale 19 / 20
    Running weight scale 20 / 20



```python
# Plot results of weight scale experiment
best_train_accs, bn_best_train_accs = [], []
best_val_accs, bn_best_val_accs = [], []
final_train_loss, bn_final_train_loss = [], []

for ws in weight_scales:
  best_train_accs.append(max(solvers[ws].train_acc_history))
  bn_best_train_accs.append(max(bn_solvers[ws].train_acc_history))
  
  best_val_accs.append(max(solvers[ws].val_acc_history))
  bn_best_val_accs.append(max(bn_solvers[ws].val_acc_history))
  
  final_train_loss.append(np.mean(solvers[ws].loss_history[-100:]))
  bn_final_train_loss.append(np.mean(bn_solvers[ws].loss_history[-100:]))
  
plt.subplot(3, 1, 1)
plt.title('Best val accuracy vs weight initialization scale')
plt.xlabel('Weight initialization scale')
plt.ylabel('Best val accuracy')
plt.semilogx(weight_scales, best_val_accs, '-o', label='baseline')
plt.semilogx(weight_scales, bn_best_val_accs, '-o', label='batchnorm')
plt.legend(ncol=2, loc='lower right')

plt.subplot(3, 1, 2)
plt.title('Best train accuracy vs weight initialization scale')
plt.xlabel('Weight initialization scale')
plt.ylabel('Best training accuracy')
plt.semilogx(weight_scales, best_train_accs, '-o', label='baseline')
plt.semilogx(weight_scales, bn_best_train_accs, '-o', label='batchnorm')
plt.legend()

plt.subplot(3, 1, 3)
plt.title('Final training loss vs weight initialization scale')
plt.xlabel('Weight initialization scale')
plt.ylabel('Final training loss')
plt.semilogx(weight_scales, final_train_loss, '-o', label='baseline')
plt.semilogx(weight_scales, bn_final_train_loss, '-o', label='batchnorm')
plt.legend()
plt.gca().set_ylim(1.0, 3.5)

plt.gcf().set_size_inches(10, 15)
plt.show()
```


![png](./Batchnormalization/output_19_0.png)


# Question:
Describe the results of this experiment, and try to give a reason why the experiment gave the results that it did.

# Answer:
Se puede ver en estos experimentos que con batchnorn la inicializacion de los pesos es mucho mas robusta y se logra generalizar mas.