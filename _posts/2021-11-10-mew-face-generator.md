---
excerpt: generate synthetic cat faces with convolutional variational autoencoder
author_profile: true
title:  "Cat Face Generator"
categories:
  - data science
tags:
  - regression
  - machine learning
  - data science
header:
  overlay_image: /assets/images/mew.jpg
  teaser: /assets/images/mew.jpg
  overlay_filter: 0.5
---

```python
from IPython import display

import glob
import imageio
import matplotlib.pyplot as plt
import numpy as np
from PIL import Image
import PIL
import tensorflow as tf
import tensorflow_probability as tfp
import time
```


```python
batch_size = 64
img_height = 64
img_width = 64
```

# Data Preparation


```python
imgs = []
for im_path in glob.glob("cats/*.jpg"):
     img = imageio.imread(im_path)
     img = np.asarray(Image.fromarray(img).resize((img_height, img_width)))
     imgs.append(img)
```


```python
len(imgs)
```




    15747




```python
all_data = np.asarray(imgs)
all_data.shape, all_data.max(), all_data.min()
```




    ((15747, 64, 64, 3), 255, 0)




```python
all_data = (all_data/255).astype('float32')
# print some images
fig = plt.figure(figsize=(16,10))
n_rows = 3
n_cols = 5
for i in range(15):
    plt.subplot(n_rows, n_cols, i + 1)
    plt.imshow(all_data[i])
```


    
![png](/assets/images/mew/output_6_0.png)
    



```python
all_data = np.asarray(imgs)
all_data = (all_data/255).astype('float32')
idx = np.random.permutation(len(all_data))
train_images = all_data[idx[:10000]]
test_images = all_data[idx[10000:]]
```


```python
train_dataset = (tf.data.Dataset.from_tensor_slices(train_images).batch(batch_size))
test_dataset = (tf.data.Dataset.from_tensor_slices(test_images).batch(batch_size))
```

# Define models


```python
class CVAE(tf.keras.Model):
  """Convolutional variational autoencoder."""

  def __init__(self, latent_dim = 8):
    super(CVAE, self).__init__()
    self.latent_dim = latent_dim
    self.encoder = tf.keras.Sequential([
            tf.keras.layers.InputLayer(input_shape=(img_height, img_width, 3)),
            tf.keras.layers.Conv2D(
                filters=32, kernel_size=3, strides=(2, 2), activation='relu'),
            tf.keras.layers.Conv2D(
                filters=64, kernel_size=3, strides=(2, 2), activation='relu'),
            tf.keras.layers.Flatten(),
            # No activation
            tf.keras.layers.Dense(latent_dim + latent_dim)])

    self.decoder = tf.keras.Sequential([
            tf.keras.layers.InputLayer(input_shape=(latent_dim,)),
            tf.keras.layers.Dense(units=16*16*32, activation=tf.nn.relu),
            tf.keras.layers.Reshape(target_shape=(16, 16, 32)),
            tf.keras.layers.Conv2DTranspose(
                filters=64, kernel_size=3, strides=2, padding='same',
                activation='relu'),
                
            tf.keras.layers.Conv2DTranspose(
                filters=32, kernel_size=3, strides=2, padding='same',
                activation='relu'),
            # No activation
            tf.keras.layers.Conv2DTranspose(
                filters=3, kernel_size=3, strides=1, padding='same')])

  @tf.function
  def sample(self, eps=None):
    if eps is None:
      eps = tf.random.normal(shape=(100, self.latent_dim))
    return self.decode(eps, apply_sigmoid=True)

  def encode(self, x):
    encoded = self.encoder(x)
    mean, logvar = tf.split(encoded, num_or_size_splits=2, axis=1)
    return mean, logvar

  def reparameterize(self, mean, logvar):
    eps = tf.random.normal(shape=mean.shape)
    return eps * tf.exp(logvar * .5) + mean

  def decode(self, z, apply_sigmoid=False):
    logits = self.decoder(z)
    if apply_sigmoid:
      probs = tf.sigmoid(logits)
      return probs
    return logits
```


```python
optimizer = tf.keras.optimizers.Adam(1e-3)


def log_normal_pdf(sample, mean, logvar, raxis=1):
  log2pi = tf.math.log(2. * np.pi)
  return tf.reduce_sum(
      -.5 * ((sample - mean) ** 2. * tf.exp(-logvar) + logvar + log2pi),
      axis=raxis)


def compute_loss(model, x):
  mean, logvar = model.encode(x)

  z = model.reparameterize(mean, logvar)

  x_logit = model.decode(z)

  cross_ent = tf.nn.sigmoid_cross_entropy_with_logits(logits=x_logit, labels=x)
  logpx_z = -tf.reduce_sum(cross_ent, axis=[1, 2, 3])
  logpz = log_normal_pdf(z, 0., 0.)
  logqz_x = log_normal_pdf(z, mean, logvar)
  return -tf.reduce_mean(logpx_z + logpz - logqz_x)


@tf.function
def train_step(model, x, optimizer):
  """Executes one training step and returns the loss.

  This function computes the loss and gradients, and uses the latter to
  update the model's parameters.
  """
  with tf.GradientTape() as tape:
    loss = compute_loss(model, x)
  gradients = tape.gradient(loss, model.trainable_variables)
  optimizer.apply_gradients(zip(gradients, model.trainable_variables))
  return -loss
```


```python
def generate_and_save_images(model, epoch, test_sample):
  mean, logvar = model.encode(test_sample)
  z = model.reparameterize(mean, logvar)
# print(z)
  predictions = model.sample(z)
  fig = plt.figure(figsize=(4, 4))

  for i in range(predictions.shape[0]):
    plt.subplot(4, 4, i + 1)
    plt.imshow(predictions[i, :, :])
    plt.axis('off')

  # tight_layout minimizes the overlap between 2 sub-plots
  plt.savefig('image_at_epoch_{:04d}.png'.format(epoch))
  plt.show()
```


```python
epochs = 38
# set the dimensionality of the latent space to a plane for visualization later
latent_dim = 128
num_examples_to_generate = 16

# Pick a sample of the test set for generating output images
assert batch_size >= num_examples_to_generate
for test_batch in test_dataset.take(1):
  test_sample = test_batch[0:num_examples_to_generate, :, :, :]
```


```python
# keeping the random vector constant for generation (prediction) so
# it will be easier to see the improvement.
random_vector_for_generation = tf.random.normal(
    shape=[num_examples_to_generate, latent_dim])
model = CVAE(latent_dim)
train_hist = []
test_hist = []
generate_and_save_images(model, 0, test_sample)
```


    
![png](/assets/images/mew/output_14_0.png)
    



```python
cur_epoch = 1
```


```python
for epoch in range(cur_epoch, epochs + 1):
  start_time = time.time()
  for train_x in train_dataset:
    train_elbo = train_step(model, train_x, optimizer)
  end_time = time.time()
  train_hist.append(train_elbo)
  loss = tf.keras.metrics.Mean()
  for test_x in test_dataset:
    loss(compute_loss(model, test_x))
  elbo = -loss.result()
  test_hist.append(elbo)
  display.clear_output(wait=False)
  print('Epoch: {}, Test set ELBO: {}, time elapse for current epoch: {}'
        .format(epoch, elbo, end_time - start_time))
  generate_and_save_images(model, epoch, test_sample)
```

    Epoch: 38, Test set ELBO: -6914.81396484375, time elapse for current epoch: 5.114240884780884
    


    
![png](/assets/images/mew/output_16_1.png)
    



   



```python
plt.plot(train_hist)
plt.plot(test_hist)
```




    [<matplotlib.lines.Line2D at 0x7fe591953e10>]




    
![png](/assets/images/mew/output_17_1.png)
    



```python
anim_file = 'cvae.gif'

with imageio.get_writer(anim_file, mode='I') as writer:
  filenames = glob.glob('image*.png')
  filenames = sorted(filenames)
  for filename in filenames:
    image = imageio.imread(filename)
    writer.append_data(image)
  image = imageio.imread(filename)
  writer.append_data(image)
```


```python
import tensorflow_docs.vis.embed as embed
embed.embed_file(anim_file)
```








```python
def display_image(epoch_no):
  return PIL.Image.open('image_at_epoch_{:04d}.png'.format(epoch_no))
```


```python
plt.imshow(display_image(epochs))
plt.axis('off')  # Display images
```

# Test generation


```python
n_rowcol = 6
z = model.reparameterize(np.zeros((n_rowcol * n_rowcol,latent_dim)).astype('float32'), 
                         np.ones((n_rowcol * n_rowcol, latent_dim)).astype('float32'))
predictions = model.sample(z)
fig = plt.figure(figsize=(16, 16))

for i in range(predictions.shape[0]):
  plt.subplot(n_rowcol, n_rowcol, i + 1)
  plt.imshow(predictions[i, :, :])
  plt.axis('off')

  # tight_layout minimizes the overlap between 2 sub-plots
plt.savefig('image_at_epoch_{:04d}.png'.format(epoch))
plt.show()
```


    
![png](/assets/images/mew/output_23_0.png)
    


# Test Reconstruction


```python
x = test_images[:15]
mean, logvar = model.encode(x)

z = model.reparameterize(mean, logvar)

x_logit = model.sample(z)

# print some images
fig = plt.figure(figsize=(16,10))
n_rows = 3
n_cols = 5
for i in range(15):
    plt.subplot(n_rows, n_cols, i + 1)
    plt.imshow(x_logit[i] )
```


    
![png](/assets/images/mew/output_25_0.png)
    



```python
fig = plt.figure(figsize=(16,10))
n_rows = 3
n_cols = 5
for i in range(15):
    plt.subplot(n_rows, n_cols, i + 1)
    plt.imshow(x[i])
```


    
![png](/assets/images/mew/output_26_0.png)
    



```python

```