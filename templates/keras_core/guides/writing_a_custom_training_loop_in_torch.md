# Writing a training loop from scratch in PyTorch

**Author:** [fchollet](https://twitter.com/fchollet)<br>
**Date created:** 2023/06/25<br>
**Last modified:** 2023/06/25<br>
**Description:** Writing low-level training & evaluation loops in PyTorch.


<img class="k-inline-icon" src="https://colab.research.google.com/img/colab_favicon.ico"/> [**View in Colab**](https://colab.research.google.com/github/keras-team/keras-io/blob/master/guides/ipynb/keras_core/writing_a_custom_training_loop_in_torch.ipynb)  <span class="k-dot">•</span><img class="k-inline-icon" src="https://github.com/favicon.ico"/> [**GitHub source**](https://github.com/keras-team/keras-io/blob/master/guides/keras_core/writing_a_custom_training_loop_in_torch.py)



---
## Setup


```python
import os

# This guide can only be run with the torch backend.
os.environ["KERAS_BACKEND"] = "torch"

import torch
import keras_core as keras
import numpy as np
```

<div class="k-default-codeblock">
```
Using PyTorch backend.

```
</div>
---
## Introduction

Keras provides default training and evaluation loops, `fit()` and `evaluate()`.
Their usage is covered in the guide
[Training & evaluation with the built-in methods](/keras_core/guides/training_with_built_in_methods/).

If you want to customize the learning algorithm of your model while still leveraging
the convenience of `fit()`
(for instance, to train a GAN using `fit()`), you can subclass the `Model` class and
implement your own `train_step()` method, which
is called repeatedly during `fit()`.

Now, if you want very low-level control over training & evaluation, you should write
your own training & evaluation loops from scratch. This is what this guide is about.

---
## A first end-to-end example

To write a custom training loop, we need the following ingredients:

- A model to train, of course.
- An optimizer. You could either use a `keras_core.optimizers` optimizer,
or a native PyTorch optimizer from `torch.optim`.
- A loss function. You could either use a `keras_core.losses` loss,
or a native PyTorch loss from `torch.nn`.
- A dataset. You could use any format: a `tf.data.Dataset`,
a PyTorch `DataLoader`, a Python generator, etc.

Let's line them up. We'll use torch-native objects in each case --
except, of course, for the Keras model.

First, let's get the model and the MNIST dataset:


```python

# Let's consider a simple MNIST model
def get_model():
    inputs = keras.Input(shape=(784,), name="digits")
    x1 = keras.layers.Dense(64, activation="relu")(inputs)
    x2 = keras.layers.Dense(64, activation="relu")(x1)
    outputs = keras.layers.Dense(10, name="predictions")(x2)
    model = keras.Model(inputs=inputs, outputs=outputs)
    return model


# Create load up the MNIST dataset and put it in a torch DataLoader
# Prepare the training dataset.
batch_size = 32
(x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()
x_train = np.reshape(x_train, (-1, 784)).astype("float32")
x_test = np.reshape(x_test, (-1, 784)).astype("float32")
y_train = keras.utils.to_categorical(y_train)
y_test = keras.utils.to_categorical(y_test)

# Reserve 10,000 samples for validation.
x_val = x_train[-10000:]
y_val = y_train[-10000:]
x_train = x_train[:-10000]
y_train = y_train[:-10000]

# Create torch Datasets
train_dataset = torch.utils.data.TensorDataset(
    torch.from_numpy(x_train), torch.from_numpy(y_train)
)
val_dataset = torch.utils.data.TensorDataset(
    torch.from_numpy(x_val), torch.from_numpy(y_val)
)

# Create DataLoaders for the Datasets
train_dataloader = torch.utils.data.DataLoader(
    train_dataset, batch_size=batch_size, shuffle=True
)
val_dataloader = torch.utils.data.DataLoader(
    val_dataset, batch_size=batch_size, shuffle=False
)
```

Next, here's our PyTorch optimizer and our PyTorch loss function:


```python
# Instantiate a torch optimizer
model = get_model()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

# Instantiate a torch loss function
loss_fn = torch.nn.CrossEntropyLoss()
```

Let's train our model using mini-batch gradient with a custom training loop.

Calling `loss.backward()` on a loss tensor triggers backpropagation.
Once that's done, your optimizer is magically aware of the gradients for each variable
and can update its variables, which is done via `optimizer.step()`.
Tensors, variables, optimizers are all interconnected to one another via hidden global state.
Also, don't forget to call `model.zero_grad()` before `loss.backward()`, or you won't
get the right gradients for your variables.

Here's our training loop, step by step:

- We open a `for` loop that iterates over epochs
- For each epoch, we open a `for` loop that iterates over the dataset, in batches
- For each batch, we call the model on the input data to retrive the predictions,
then we use them to compute a loss value
- We call `loss.backward()` to
- Outside the scope, we retrieve the gradients of the weights
of the model with regard to the loss
- Finally, we use the optimizer to update the weights of the model based on the
gradients


```python
epochs = 3
for epoch in range(epochs):
    for step, (inputs, targets) in enumerate(train_dataloader):
        # Forward pass
        logits = model(inputs)
        loss = loss_fn(logits, targets)

        # Backward pass
        model.zero_grad()
        loss.backward()

        # Optimizer variable updates
        optimizer.step()

        # Log every 100 batches.
        if step % 100 == 0:
            print(
                f"Training loss (for 1 batch) at step {step}: {loss.detach().numpy():.4f}"
            )
            print(f"Seen so far: {(step + 1) * batch_size} samples")
```

<div class="k-default-codeblock">
```
Training loss (for 1 batch) at step 0: 164.5633
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 7.2865
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 4.0144
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 1.5672
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 2.5375
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.4000
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 2.9213
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.7962
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.5127
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.4609
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 1.0465
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 1.5726
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.3018
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.6223
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.6646
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.5086
Seen so far: 48032 samples
Training loss (for 1 batch) at step 0: 0.6439
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 0.8085
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 0.1590
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 1.3015
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.5472
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.4914
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.4737
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.5811
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.6876
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.4832
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.1450
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.1185
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.4151
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.5973
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.3626
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.3810
Seen so far: 48032 samples
Training loss (for 1 batch) at step 0: 0.1599
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 0.3633
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 0.1922
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.4137
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.4604
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.2268
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.0840
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.4261
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.0633
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 1.2083
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.5303
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.0430
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.2571
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.2738
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.1873
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.1968
Seen so far: 48032 samples

```
</div>
As an alternative, let's look at what the loop looks like when using a Keras optimizer
and a Keras loss function.

Important differences:

- You retrieve the gradients for the variables via `v.value.grad`,
called on each trainable variable.
- You update your variables via `optimizer.apply()`, which must be
called in a `torch.no_grad()` scope.

**Also, a big gotcha:** while all NumPy/TensorFlow/JAX/Keras APIs
as well as Python `unittest` APIs use the argument order convention
`fn(y_true, y_pred)` (reference values first, predicted values second),
PyTorch actually uses `fn(y_pred, y_true)` for its losses.
So make sure to invert the order of `logits` and `targets`.


```python
model = get_model()
optimizer = keras.optimizers.Adam(learning_rate=1e-3)
loss_fn = keras.losses.CategoricalCrossentropy(from_logits=True)

for epoch in range(epochs):
    print(f"\nStart of epoch {epoch}")
    for step, (inputs, targets) in enumerate(train_dataloader):
        # Forward pass
        logits = model(inputs)
        loss = loss_fn(targets, logits)

        # Backward pass
        model.zero_grad()
        trainable_weights = [v for v in model.trainable_weights]

        # Call torch.Tensor.backward() on the loss to compute gradients
        # for the weights.
        loss.backward()
        gradients = [v.value.grad for v in trainable_weights]

        # Update weights
        with torch.no_grad():
            optimizer.apply(gradients, trainable_weights)

        # Log every 100 batches.
        if step % 100 == 0:
            print(
                f"Training loss (for 1 batch) at step {step}: {loss.detach().numpy():.4f}"
            )
            print(f"Seen so far: {(step + 1) * batch_size} samples")
```

    
<div class="k-default-codeblock">
```
Start of epoch 0
Training loss (for 1 batch) at step 0: 122.0479
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 3.1449
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 2.0521
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.7800
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.7696
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.6281
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 1.2518
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 1.4177
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.8149
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 1.0081
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.5257
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.2828
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.7151
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.6575
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.4036
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.7787
Seen so far: 48032 samples
```
</div>
    
<div class="k-default-codeblock">
```
Start of epoch 1
Training loss (for 1 batch) at step 0: 0.2625
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 0.2798
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 0.5291
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.6563
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.2270
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.3130
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.4169
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.1511
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.8881
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.3289
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.3924
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.4998
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.6184
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.2083
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.2224
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.5366
Seen so far: 48032 samples
```
</div>
    
<div class="k-default-codeblock">
```
Start of epoch 2
Training loss (for 1 batch) at step 0: 0.6056
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 0.2684
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 0.2667
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.4145
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.6769
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.1293
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.0555
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.4172
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.2016
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.0636
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.4378
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.2096
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.1127
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.2296
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.2888
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.6187
Seen so far: 48032 samples

```
</div>
---
## Low-level handling of metrics

Let's add metrics monitoring to this basic training loop.

You can readily reuse built-in Keras metrics (or custom ones you wrote) in such training
loops written from scratch. Here's the flow:

- Instantiate the metric at the start of the loop
- Call `metric.update_state()` after each batch
- Call `metric.result()` when you need to display the current value of the metric
- Call `metric.reset_state()` when you need to clear the state of the metric
(typically at the end of an epoch)

Let's use this knowledge to compute `CategoricalAccuracy` on training and
validation data at the end of each epoch:


```python
# Get a fresh model
model = get_model()

# Instantiate an optimizer to train the model.
optimizer = keras.optimizers.Adam(learning_rate=1e-3)
# Instantiate a loss function.
loss_fn = keras.losses.CategoricalCrossentropy(from_logits=True)

# Prepare the metrics.
train_acc_metric = keras.metrics.CategoricalAccuracy()
val_acc_metric = keras.metrics.CategoricalAccuracy()
```

Here's our training & evaluation loop:


```python
for epoch in range(epochs):
    print(f"\nStart of epoch {epoch}")
    for step, (inputs, targets) in enumerate(train_dataloader):
        # Forward pass
        logits = model(inputs)
        loss = loss_fn(targets, logits)

        # Backward pass
        model.zero_grad()
        trainable_weights = [v for v in model.trainable_weights]

        # Call torch.Tensor.backward() on the loss to compute gradients
        # for the weights.
        loss.backward()
        gradients = [v.value.grad for v in trainable_weights]

        # Update weights
        with torch.no_grad():
            optimizer.apply(gradients, trainable_weights)

        # Update training metric.
        train_acc_metric.update_state(targets, logits)

        # Log every 100 batches.
        if step % 100 == 0:
            print(
                f"Training loss (for 1 batch) at step {step}: {loss.detach().numpy():.4f}"
            )
            print(f"Seen so far: {(step + 1) * batch_size} samples")

    # Display metrics at the end of each epoch.
    train_acc = train_acc_metric.result()
    print(f"Training acc over epoch: {float(train_acc):.4f}")

    # Reset training metrics at the end of each epoch
    train_acc_metric.reset_state()

    # Run a validation loop at the end of each epoch.
    for x_batch_val, y_batch_val in val_dataloader:
        val_logits = model(x_batch_val, training=False)
        # Update val metrics
        val_acc_metric.update_state(y_batch_val, val_logits)
    val_acc = val_acc_metric.result()
    val_acc_metric.reset_state()
    print(f"Validation acc: {float(val_acc):.4f}")

```

    
<div class="k-default-codeblock">
```
Start of epoch 0
Training loss (for 1 batch) at step 0: 77.5892
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 5.1572
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 3.3523
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.4230
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.5259
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 1.0390
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.8866
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.5704
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.4057
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 1.2603
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.5393
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 1.7498
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.1861
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.3598
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.7257
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.2266
Seen so far: 48032 samples
Training acc over epoch: 0.8046
Validation acc: 0.8845
```
</div>
    
<div class="k-default-codeblock">
```
Start of epoch 1
Training loss (for 1 batch) at step 0: 0.3444
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 1.0648
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 0.6529
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.5622
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.1252
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.4107
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.8118
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.7214
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.4464
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.3732
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.0978
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.3165
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.5874
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.2385
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.5096
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.2982
Seen so far: 48032 samples
Training acc over epoch: 0.8920
Validation acc: 0.9143
```
</div>
    
<div class="k-default-codeblock">
```
Start of epoch 2
Training loss (for 1 batch) at step 0: 0.5947
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 0.0895
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 0.1536
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.1926
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.1102
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.6353
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.2784
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.4402
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.4561
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.2178
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.3840
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.4225
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.3519
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.1569
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.4598
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.1951
Seen so far: 48032 samples
Training acc over epoch: 0.9090
Validation acc: 0.9279

```
</div>
---
## Low-level handling of losses tracked by the model

Layers & models recursively track any losses created during the forward pass
by layers that call `self.add_loss(value)`. The resulting list of scalar loss
values are available via the property `model.losses`
at the end of the forward pass.

If you want to be using these loss components, you should sum them
and add them to the main loss in your training step.

Consider this layer, that creates an activity regularization loss:


```python

class ActivityRegularizationLayer(keras.layers.Layer):
    def call(self, inputs):
        self.add_loss(1e-2 * torch.sum(inputs))
        return inputs

```

Let's build a really simple model that uses it:


```python
inputs = keras.Input(shape=(784,), name="digits")
x = keras.layers.Dense(64, activation="relu")(inputs)
# Insert activity regularization as a layer
x = ActivityRegularizationLayer()(x)
x = keras.layers.Dense(64, activation="relu")(x)
outputs = keras.layers.Dense(10, name="predictions")(x)

model = keras.Model(inputs=inputs, outputs=outputs)
```

Here's what our training loop should look like now:


```python
# Get a fresh model
model = get_model()

# Instantiate an optimizer to train the model.
optimizer = keras.optimizers.Adam(learning_rate=1e-3)
# Instantiate a loss function.
loss_fn = keras.losses.CategoricalCrossentropy(from_logits=True)

# Prepare the metrics.
train_acc_metric = keras.metrics.CategoricalAccuracy()
val_acc_metric = keras.metrics.CategoricalAccuracy()

for epoch in range(epochs):
    print(f"\nStart of epoch {epoch}")
    for step, (inputs, targets) in enumerate(train_dataloader):
        # Forward pass
        logits = model(inputs)
        loss = loss_fn(targets, logits)
        if model.losses:
            loss = loss + torch.sum(*model.losses)

        # Backward pass
        model.zero_grad()
        trainable_weights = [v for v in model.trainable_weights]

        # Call torch.Tensor.backward() on the loss to compute gradients
        # for the weights.
        loss.backward()
        gradients = [v.value.grad for v in trainable_weights]

        # Update weights
        with torch.no_grad():
            optimizer.apply(gradients, trainable_weights)

        # Update training metric.
        train_acc_metric.update_state(targets, logits)

        # Log every 100 batches.
        if step % 100 == 0:
            print(
                f"Training loss (for 1 batch) at step {step}: {loss.detach().numpy():.4f}"
            )
            print(f"Seen so far: {(step + 1) * batch_size} samples")

    # Display metrics at the end of each epoch.
    train_acc = train_acc_metric.result()
    print(f"Training acc over epoch: {float(train_acc):.4f}")

    # Reset training metrics at the end of each epoch
    train_acc_metric.reset_state()

    # Run a validation loop at the end of each epoch.
    for x_batch_val, y_batch_val in val_dataloader:
        val_logits = model(x_batch_val, training=False)
        # Update val metrics
        val_acc_metric.update_state(y_batch_val, val_logits)
    val_acc = val_acc_metric.result()
    val_acc_metric.reset_state()
    print(f"Validation acc: {float(val_acc):.4f}")
```

    
<div class="k-default-codeblock">
```
Start of epoch 0
Training loss (for 1 batch) at step 0: 125.4331
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 5.3620
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 2.1645
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 4.1102
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 3.0057
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.7266
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 1.3168
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 2.0576
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.9830
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.6269
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.2761
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.2517
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.4546
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.8293
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.5323
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.6470
Seen so far: 48032 samples
Training acc over epoch: 0.8273
Validation acc: 0.9000
```
</div>
    
<div class="k-default-codeblock">
```
Start of epoch 1
Training loss (for 1 batch) at step 0: 0.8858
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 0.3555
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 1.3821
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.9273
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.4524
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.4216
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.2071
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.4984
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.5847
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.3308
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.0705
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.1850
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.2787
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.8273
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.0410
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.4980
Seen so far: 48032 samples
Training acc over epoch: 0.9099
Validation acc: 0.9286
```
</div>
    
<div class="k-default-codeblock">
```
Start of epoch 2
Training loss (for 1 batch) at step 0: 0.1715
Seen so far: 32 samples
Training loss (for 1 batch) at step 100: 0.2243
Seen so far: 3232 samples
Training loss (for 1 batch) at step 200: 0.5812
Seen so far: 6432 samples
Training loss (for 1 batch) at step 300: 0.3141
Seen so far: 9632 samples
Training loss (for 1 batch) at step 400: 0.1263
Seen so far: 12832 samples
Training loss (for 1 batch) at step 500: 0.2595
Seen so far: 16032 samples
Training loss (for 1 batch) at step 600: 0.4545
Seen so far: 19232 samples
Training loss (for 1 batch) at step 700: 0.0636
Seen so far: 22432 samples
Training loss (for 1 batch) at step 800: 0.3816
Seen so far: 25632 samples
Training loss (for 1 batch) at step 900: 0.1047
Seen so far: 28832 samples
Training loss (for 1 batch) at step 1000: 0.2365
Seen so far: 32032 samples
Training loss (for 1 batch) at step 1100: 0.0787
Seen so far: 35232 samples
Training loss (for 1 batch) at step 1200: 0.4984
Seen so far: 38432 samples
Training loss (for 1 batch) at step 1300: 0.0766
Seen so far: 41632 samples
Training loss (for 1 batch) at step 1400: 0.0523
Seen so far: 44832 samples
Training loss (for 1 batch) at step 1500: 0.5910
Seen so far: 48032 samples
Training acc over epoch: 0.9332
Validation acc: 0.9208

```
</div>
That's it!
