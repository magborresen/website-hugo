---
categories: python
date: "2023-04-20"
description: Methods for separating and batching data
tags: ["python", "keras", "tensorflow", "time-series"]
author: ["me"]
title: Preparing Time Series Data For Neural Network Inputs
katex: true
draft: false
---

While working on my master's thesis I needed some processing of temporal data such that it could be used as an input in TensorFlow Keras. Here I stumbled across the *TimeseriesGenerator*, which is a function from the Keras library. However, this is no longer recommended to be used by TensorFlow.

Instead, I found another function called *timeseries_dataset_from_array*. Which takes a Numpy array and converts it into a TensorFlow dataset-type object. I would not always recommend using this, and it might be easier to just create your own function for loading and separating the data. But the *timeseries_dataset_from_array* function is super neat if you are working with a lot of data and are trying to save space. 

So the function is meant to create sliding windows over your array. The function is defined with the following parameters

```python3
tf.keras.utils.timeseries_dataset_from_array(
    data,
    targets,
    sequence_length,
    sequence_stride=1,
    sampling_rate=1,
    batch_size=128,
    shuffle=False,
    seed=None,
    start_index=None,
    end_index=None
)
```

So you of course need to give it some data. You don't necessarily have to specify the targets. For my thesis, I was working with autoencoders for reconstruction. So my targets are the same as my data. Depending on your problem type, this could be the next data point (for regression), or a label if you are trying to classify your data.

The next parameter you are going to want to set is the sequence length. This defined the number of consecutive time samples there will be in one output sample. With a caveat. If you set the sequence stride to anything lower than the sequence length, your output will be overlapped. Meaning if you set the sequence length to $256$ and you set the sequence stride to $128$, then for your first batch sample, all $256$ time samples will be unique. But for the second batch sample, there will be $128$ overlapped from the first batch sample and then $128$ unique samples. And so on.

You can use the sample rate parameter to downsample your signal if needed. If you want all data points to be included in the output set, then set it to $1$. 

Then of course set the batch size to the size of the batches you need for training. The default is $128$, meaning there will be $128$ batches of $256$ time samples each. 