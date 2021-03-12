
# MixUp augmentation for image classification

**Author:** [Sayak Paul](https://twitter.com/RisingSayak)<br>
**Date created:** 2021/03/06<br>
**Last modified:** 2021/03/06<br>


<img class="k-inline-icon" src="https://colab.research.google.com/img/colab_favicon.ico"/> [**View in Colab**](https://colab.research.google.com/github/keras-team/keras-io/blob/master/examples/vision/ipynb/mixUp.ipynb)  <span class="k-dot">•</span><img class="k-inline-icon" src="https://github.com/favicon.ico"/> [**GitHub source**](https://github.com/keras-team/keras-io/blob/master/examples/vision/mixUp.py)


**Description:** Data augmentation using the mixup technique for image classification.

---
## Introduction

_mixup_ is a *domain-agnostic* data augmentation technique proposed in [mixup: Beyond Empirical Risk Minimization](https://arxiv.org/abs/1710.09412)
by Zhang et al. It's implemented with the following formulas:

![](https://i.ibb.co/DRyHYww/image.png)

(Note that the lambda values are values with the [0, 1] range and are sampled from the
[Beta distribution](https://en.wikipedia.org/wiki/Beta_distribution).)

The technique is quite systematically named - we are literally mixing up the features and
their corresponding labels. Implementation-wise it's simple. Neural networks are prone
to [memorizing corrupt labels](https://arxiv.org/abs/1611.03530). mixup relaxes this by
combining different features with one another (same happens for the labels too) so that
a network does not get overconfident about the relationship between the features and
their labels.

mixup is specifically useful when we are not sure about selecting a set of augmentation
transforms for a given dataset, medical imaging datasets, for example. mixup can be
extended to a variety of data modalities such as computer vision, naturallanguage
processing, speech, and so on.

This example requires TensorFlow 2.4 or higher.

---
## Setup


```python
import numpy as np
import tensorflow as tf
import matplotlib.pyplot as plt
from tensorflow.keras import layers
```

---
## Prepare the dataset

In this example, we will be using the [FashionMNIST](https://research.zalando.com/welcome/mission/research-projects/fashion-mnist/) dataset. But this same recipe can
be used for other classification datasets as well.


```python
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.fashion_mnist.load_data()

x_train = x_train.astype("float32") / 255.0
x_train = np.reshape(x_train, (-1, 28, 28, 1))
y_train = tf.one_hot(y_train, 10)

x_test = x_test.astype("float32") / 255.0
x_test = np.reshape(x_test, (-1, 28, 28, 1))
y_test = tf.one_hot(y_test, 10)
```

<div class="k-default-codeblock">
```
Downloading data from https://storage.googleapis.com/tensorflow/tf-keras-datasets/train-labels-idx1-ubyte.gz
32768/29515 [=================================] - 0s 0us/step
Downloading data from https://storage.googleapis.com/tensorflow/tf-keras-datasets/train-images-idx3-ubyte.gz
26427392/26421880 [==============================] - 0s 0us/step
Downloading data from https://storage.googleapis.com/tensorflow/tf-keras-datasets/t10k-labels-idx1-ubyte.gz
8192/5148 [===============================================] - 0s 0us/step
Downloading data from https://storage.googleapis.com/tensorflow/tf-keras-datasets/t10k-images-idx3-ubyte.gz
4423680/4422102 [==============================] - 0s 0us/step

```
</div>
---
## Define hyperparameters


```python
AUTO = tf.data.AUTOTUNE
BATCH_SIZE = 64
EPOCHS = 5
```

---
## Convert the data into TensorFlow `Dataset` objects


```python
# Put aside a few samples to create our validation set
val_samples = 2000
x_val, y_val = x_train[:val_samples], y_train[:val_samples]
new_x_train, new_y_train = x_train[val_samples:], y_train[val_samples:]

train_ds_one = (
    tf.data.Dataset.from_tensor_slices((new_x_train, new_y_train))
    .shuffle(BATCH_SIZE * 100)
    .batch(BATCH_SIZE)
)
train_ds_two = (
    tf.data.Dataset.from_tensor_slices((new_x_train, new_y_train))
    .shuffle(BATCH_SIZE * 100)
    .batch(BATCH_SIZE)
)
# Because we will be mixing up the images and their corresponding labels, we will be
# combining two shuffled datasets from the same training data.
train_ds = tf.data.Dataset.zip((train_ds_one, train_ds_two))

val_ds = tf.data.Dataset.from_tensor_slices((x_val, y_val)).batch(BATCH_SIZE)

test_ds = tf.data.Dataset.from_tensor_slices((x_test, y_test)).batch(BATCH_SIZE)
```

---
## Define the mixup technique function

To perform the mixup routine, we create new virtual datasets using the training data from
the same dataset, and apply a lambda value within the [0, 1] range sampled from a [Beta distribution](https://en.wikipedia.org/wiki/Beta_distribution)
— such that, for example, `new_x = lambda * x1 + (1 - lambda) * x2` (where
`x1` and `x2` are images) and the same equation is applied to the labels as well.


```python

def sample_beta_distribution(size, concentration_0=0.2, concentration_1=0.2):
    gamma_1_sample = tf.random.gamma(shape=[size], alpha=concentration_1)
    gamma_2_sample = tf.random.gamma(shape=[size], alpha=concentration_0)
    return gamma_1_sample / (gamma_1_sample + gamma_2_sample)


def mix_up(ds_one, ds_two, alpha=0.2):
    # Unpack two datasets
    images_one, labels_one = ds_one
    images_two, labels_two = ds_two
    batch_size = tf.shape(images_one)[0]

    # Sample lambda and reshape it to do the mixup
    l = sample_beta_distribution(batch_size, 0.2, 0.2)
    x_l = tf.reshape(l, (batch_size, 1, 1, 1))
    y_l = tf.reshape(l, (batch_size, 1))

    # Perform mixup on both images and labels by combining a pair of images/labels
    # (one from each dataset) into one image/label
    images = images_one * x_l + images_two * (1 - x_l)
    labels = labels_one * y_l + labels_two * (1 - y_l)
    return (images, labels)

```

**Note** that here , we are combining two images to create a single one. Theoretically,
we can combine as many we want but that comes at an increased computation cost. In
certain cases, it may not help improve the performance as well.

---
## Visualize the new augmented dataset


```python
# First create the new dataset using our `mix_up` utility
train_ds_mu = train_ds.map(
    lambda ds_one, ds_two: mix_up(ds_one, ds_two, alpha=0.2), num_parallel_calls=AUTO
)

# Let's preview 9 samples from the dataset
sample_images, sample_labels = next(iter(train_ds_mu))
plt.figure(figsize=(10, 10))
for i, (image, label) in enumerate(zip(sample_images[:9], sample_labels[:9])):
    ax = plt.subplot(3, 3, i + 1)
    plt.imshow(image.numpy().squeeze())
    print(label.numpy().tolist())
    plt.axis("off")
```

<div class="k-default-codeblock">
```
[0.0, 0.9999974966049194, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 2.5033950805664062e-06]
[0.0, 0.0, 0.0, 0.0, 0.0, 0.11348208785057068, 0.8865178823471069, 0.0, 0.0, 0.0]
[0.0, 0.0, 0.0016847170190885663, 0.0, 0.0, 0.0, 0.0, 0.0, 0.9983152747154236, 0.0]
[0.0, 0.04897904023528099, 0.0, 0.0, 0.9510209560394287, 0.0, 0.0, 0.0, 0.0, 0.0]
[0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.9999988675117493, 1.1195215847692452e-06, 0.0]
[0.0, 0.0, 0.0, 0.9978762269020081, 0.0, 0.0, 0.0, 0.0, 0.0021237730979919434, 0.0]
[0.0, 0.0, 0.0004618267703335732, 0.9995381832122803, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
[0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
[0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]

```
</div>
![png](/img/examples/vision/mixUp/mixUp_15_1.png)


---
## Model building


```python

def get_training_model():
    model = tf.keras.Sequential(
        [
            layers.Conv2D(16, (5, 5), activation="relu", input_shape=(28, 28, 1)),
            layers.MaxPooling2D(pool_size=(2, 2)),
            layers.Conv2D(32, (5, 5), activation="relu"),
            layers.MaxPooling2D(pool_size=(2, 2)),
            layers.Dropout(0.2),
            layers.GlobalAvgPool2D(),
            layers.Dense(128, activation="relu"),
            layers.Dense(10, activation="softmax"),
        ]
    )
    return model

```

For the sake of reproducibility, we serialize the initial random weights of our shallow
network.


```python
initial_model = get_training_model()
initial_model.save_weights("initial_weights.h5")
```

---
## 1. Train the model with the mixed up dataset


```python
model = get_training_model()
model.load_weights("initial_weights.h5")
model.compile(loss="categorical_crossentropy", optimizer="adam", metrics=["accuracy"])
model.fit(train_ds_mu, validation_data=val_ds, epochs=EPOCHS)
_, test_acc = model.evaluate(test_ds)
print("Test accuracy: {:.2f}%".format(test_acc * 100))
```

<div class="k-default-codeblock">
```
Epoch 1/5
907/907 [==============================] - 36s 4ms/step - loss: 1.4261 - accuracy: 0.5208 - val_loss: 0.6958 - val_accuracy: 0.7430
Epoch 2/5
907/907 [==============================] - 3s 3ms/step - loss: 0.9633 - accuracy: 0.7194 - val_loss: 0.5709 - val_accuracy: 0.8030
Epoch 3/5
907/907 [==============================] - 3s 4ms/step - loss: 0.8721 - accuracy: 0.7609 - val_loss: 0.4988 - val_accuracy: 0.8270
Epoch 4/5
907/907 [==============================] - 3s 3ms/step - loss: 0.8323 - accuracy: 0.7770 - val_loss: 0.4683 - val_accuracy: 0.8485
Epoch 5/5
907/907 [==============================] - 3s 3ms/step - loss: 0.7964 - accuracy: 0.7912 - val_loss: 0.4587 - val_accuracy: 0.8435
157/157 [==============================] - 0s 2ms/step - loss: 0.4814 - accuracy: 0.8299
Test accuracy: 82.99%

```
</div>
---
## 2. Train the model *without* the mixed up dataset


```python
model = get_training_model()
model.load_weights("initial_weights.h5")
model.compile(loss="categorical_crossentropy", optimizer="adam", metrics=["accuracy"])
# Notice that we are NOT using the mixed up dataset here
model.fit(train_ds_one, validation_data=val_ds, epochs=EPOCHS)
_, test_acc = model.evaluate(test_ds)
print("Test accuracy: {:.2f}%".format(test_acc * 100))
```

<div class="k-default-codeblock">
```
Epoch 1/5
907/907 [==============================] - 3s 3ms/step - loss: 1.1834 - accuracy: 0.5570 - val_loss: 0.6639 - val_accuracy: 0.7670
Epoch 2/5
907/907 [==============================] - 3s 3ms/step - loss: 0.6553 - accuracy: 0.7547 - val_loss: 0.5536 - val_accuracy: 0.7900
Epoch 3/5
907/907 [==============================] - 3s 3ms/step - loss: 0.5712 - accuracy: 0.7868 - val_loss: 0.4954 - val_accuracy: 0.8240
Epoch 4/5
907/907 [==============================] - 3s 3ms/step - loss: 0.5169 - accuracy: 0.8101 - val_loss: 0.4742 - val_accuracy: 0.8310
Epoch 5/5
907/907 [==============================] - 3s 3ms/step - loss: 0.4799 - accuracy: 0.8239 - val_loss: 0.4324 - val_accuracy: 0.8460
157/157 [==============================] - 0s 2ms/step - loss: 0.4654 - accuracy: 0.8357
Test accuracy: 83.57%

```
</div>
Readers are encouraged to try out mixup on different datasets from different domains and
experiment with the lambda parameter. You are strongly advised to check out the
[original paper](https://arxiv.org/abs/1710.09412) as well - the authors present several ablation studies on mixup
showing how it can improve generalization, as well as show their results of combining
more than two images to create a single one.

---
## Notes

* With mixup, you can create synthetic examples — especially when you lack a large
dataset - without incurring high computational costs.
* [Label smoothing](https://www.pyimagesearch.com/2019/12/30/label-smoothing-with-keras-tensorflow-and-deep-learning/) and mixup usually do not work well together because label smoothing
already modifies the hard labels by some factor.
* mixup does not work well when you are using [Supervised Contrastive
Learning](https://arxiv.org/abs/2004.11362) (SCL) since SCL expects the true labels
during its pre-training phase.
* A few other benefits of mixup include (as described in the [paper](https://arxiv.org/abs/1710.09412)) robustness to
adversarial examples and stabilized GAN (Generative Adversarial Networks) training.
* There are a number of data augmentation techniques that extend mixup such as
[CutMix](https://arxiv.org/abs/1905.04899) and [AugMix](https://arxiv.org/abs/1912.02781).