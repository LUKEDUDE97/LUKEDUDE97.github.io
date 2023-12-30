---
layout: post
title: Plot Digitizer - Produce Random Synthetic Images
date: 2023-12-30 10:20 +0100

categories: [Zero-Project, Plot Digitizer]
tags: [Plot Digitizer, Pytorch, Memory Leakage]

post_description: Create random diagram, format it into torch dataset with labeling and trying to extract information later via transformers DETR. A memory leakage issue was dealing in this note BTW.
---

## I. Objectives 

To build a generalized model for extracting accurate digits from plots, like line plots, dot-line plots, scatters, vertical and horizontal plots and of course pie charts, etc. Here are some examples. 

![Sample plots of different types](/localdata/assets/ZeroProject/PlotSamples.PNG){: .shadow width="740"}
_Sample plots of different types_

In this work, we intend to solve this problem in a sequence-based manner. That is to say, based on deformable detection transformer (DETR) and other important query-based methods, the input images will be converted into sequential digits with variable length. Yet it is still too soon to get into details. As for this first log, we mainly focus on how to produce random sythetic plots for our dataset. 


## II. Draw by Matplotlib

First we try to accomplish three different typical types: bar, scatter and dot-line. We here introduce matplotlib for random drawing.

```python 
from matplotlib import pyplot as plt
```

We don't want to give too much pressure on our first test model. Therefore all our images are set into a fixed size via:

```python 
plt.rcParams['figure.figsize'] = (5, 5)
```

Thus the plots' general appearance are unified and all in a matplotlib style. Later we will try to introduce a public dataset, like FigureQA, to boost the generalization ability. But today we just want to know if it's work at all.

+ bar plot 

    ```python
    bar_count = np.random.randint(2, 10)
    x = np.linspace(0, 1, bar_count)
    y = np.random.random(bar_count)

    plt.figure()

    plt.bar(x, y, width=1/bar_count)
    plt.close()
    ```

+ scatter plot

    ```python 
    scatter_counts = np.random.randint(9, 15)
    x = np.random.random(scatter_counts)
    y = np.random.random(scatter_counts)

    plt.figure()
    plt.xlim(0, 1)
    plt.ylim(0, 1)
    colors = np.random.rand(scatter_counts)

    plt.scatter(x, y, s=80, c=colors, alpha=1)
    plt.close()
    ```

+ dot-line plot

    ```python 
    keypoint_counts = np.random.randint(3, 10)
    x = np.linspace(0.1, 0.9, keypoint_counts)
    y = np.random.random(keypoint_counts)

    plt.figure()
    plt.xlim(0, 1)
    plt.ylim(0, 1)
    
    plt.plot(x, y, marker="*", ms=10, linewidth=4.0, mfc="4CAF50", alpha=0.6)
    plt.close()
    ```

The so-called digits from the plot is composed by several keypoints which consist of `x, y` representing the x-, y- axis value in percentage (0 ~ 1, so far it is enough). Considering it is genrally a continuous variable on x-aixs in dot-line plot and scatters, we adopted `ply.xlim` to take all part of x-axis into use. If we got a real value of start-point and end-point on x-, y-axis, it would be easy for us to directly translate the percentage value into some real content in the real world. But in the bar plot, it is discrete on x-axis. Only the predicted y percentage-value matters here actually. A much more sophisticated reasoning on x-axis and plot-legend will be needed at the end. 

The number of keypoints equals to the final predicted sequnence's length, it is variable and defined by `_counts`.

![bar, scatters and dot-line](/localdata/assets/ZeroProject/PlotSamples_plt.png){: .shadow width="740"}
_bar, scatters and dot-line_

Ok, so we can produce random synthetic images now. They are all unified into similiar form and style. It is not perfect but enough for our start test. Let's put them into the sub-class of `Dataset` from torch then. 

## III. Dataset Class

### a. DataLoader Introduction

![DataLoader Pic](/localdata/assets/ZeroProject/DataLoader.png){: .w-50 .right}

We need a customized `Dataset` sub-class for our data object. 

> All dataset that represent **a map from keys to data samples** should subclass it. All subclasses should overwrite `__getitem__()`, supporting fetching a data sample for a given key. Subclasses could also optionally overwrite `__len__()`, which is expected to return the size of the dataset by many `Sampler` implementations and the default options of `DataLoader`.

``` python 
class DataLoader(object):
    ...

    def __next__(self):
        if self.num_workers == 0:  
            indices = next(self.sample_iter)  
            batch = self.collate_fn([self.dataset[i] for i in indices]) 
            if self.pin_memory:
                batch = _utils.pin_memory.pin_memory_batch(batch)
            return batch
```

This is a part of source code from `DataLoader.__next__`:
    
1. DataLoader by default constructs an index sampler to yields these integer indices (iterable, thus can fetch via `next`) which were later used to fetch class instances from Dataset subclass.
2. `__getitem__` makes it possible to fetch class instancelike it was a list, which is an built-in python function and defined in subclass. 
3. Everytime from `dataset[idx]` we can only get one single sample, normally it returns a tuple which includes a pair of image and label or bbox. The output has to be reorganized by `collate_fn` in batch style. 
4. Finally, pass the batch data of this iteration from CPU to GPU via `pin_memory_batch`.

Thus, what we have to do here is to build a customized `Dataset` sub-class and define the function, `__getitem__` and `collate_fn`, as needed. 

By the way, that is an old version of  `DataLoader` from torch and we only select the section when the `num_works` was set to zero. It is easy to illustrate from that way. And the code from newest stable version (1.13.0) is like: 

``` python 
class _SingleProcessDataLoaderIter(_BaseDataLoaderIter):
    ...

    def _next_data(self):
        index = self._next_index()  # may raise StopIteration
        data = self._dataset_fetcher.fetch(index)  # may raise StopIteration
        if self._pin_memory:
            data = _utils.pin_memory.pin_memory(data, self._pin_memory_device)
        return data
```


### b. Generate Data Item 

``` python 
    def __getitem__(self, index):
        seed = index + self.start_seed
        np.random.seed(seed)
```
Index corresponds to the indice we got from sampler and enables us to instantiate an exclusive data sample. By combining it with our original seed value, we got a random different seed every time which means plots with different appearance. 

``` python 
        bbox = np.concatenate([
            x[:, None], y[:, None], np.zeros((x.shape[0], 1)), np.zeros((x.shape[0], 1))
        ], aixs=1)
```

As it mentioned before, we summerized the plot by numerous keypoints, with `x-` and `y-axis` value. Yet here we not only include the keypoint axis value but also a `weight` and `height` for bbox. It is natural because our inspiration is from general query-based object detection algorithm. We leave it to be temporarily. It has nothing to do with the training yet since we set them all into zero and L1 loss only conduct on coordinates. But later, we will have some adjustment on it to introduce some geometrical constraint into this format and dilate the general unit of these keypoints. 

***

#### A Memory Leakage

Some unexpected issue happened here when we were trying to turn our `PILImage` into Tensor: (Version: Python - 3.6.5, Numpy - 1.19.5, Torch - 1.10.1)

``` python 
    PILImage = Image.open(buff).convert('RGB').resize((480, 480))
    TensorImage = ToTensor()(PILImage)          # memory leakage
```

![Memory Leakage on ToTensor](/localdata/assets/ZeroProject/MemoryLeakageOnToTensor.PNG){: .shadow}
_Memory Leakage on ToTensor_

A memory leakage happened here. Because of this, the memory usage rapidly piled up and soon shut down automatically. **Memory Profile** was used here to seek the leak point. It is quite powerful. It shows that every time when the `ToTensor()` were used, it spare out some additional space in memory and can not be automatically released after *the reference count back to zero*, or what we say, finished (I'm a starter on python's garabage collection mechanism).

We later go to the detail of class `ToTensor`:

> Converts a PIL Image or numpy.ndarray (H x W x C) in the range [0, 255] to a torch.FloatTensor of shape (C x H x W) in the range [0.0, 1.0] if the PIL Image belongs to one of the modes (L, LA, P, I, F, RGB, YCbCr, RGBA, CMYK, 1) or if the numpy.ndarray has dtype = np.uint8

According to that, we reimplement the transformation trying to find where the leakage exactly happened.

``` python 
    img_arr = PILImage.astype(np.float32)   
    img_arr = img_arr / 255.                    # memory leakage
    img_arr = torch.from_numpy(img_arr)
    img_arr = img_arr.permute(2, 0, 1)
```

![Memory Leakage on Numpy](/localdata/assets/ZeroProject/MemoryLeakageOnNumpy.PNG){: .shadow}
_Memory Leakage during Calculation on array_

It is nature for numpy. Yet what we expect, automatically memory release after that, never happened. We even tried to manually release it via `gc.collect()` but turned out to be useless. Till then, we still don't know why. Therefore, we implemented it in a multiprocessing style at the end. 

``` python
    def __getitem__(self, index):
        seed = index + self.start_seed
        with Pool(1) as p:
            return p.apply(self._generate_dataset, [seed])
```

The plot sample will be generated and transformed to Tensor in `_generate_dataset`, and the memory will be automatically released after the process were terminated. It works but costs more time for training.

You may encounter following errors for conducting multiprocess here, a detailed discussion on python multiprocessing and `apply()`, `map()` function is present in my another post where you can find solutions. 

> TypeError: `run()` arguement after * must be an iterable, not int
{: .prompt-danger}

> ValueError: not enough values to unpack (expected *, got 1)
{: .prompt-danger}

<!-- > Regarding apply vs map:
>
>>`Pool.apply(f, args)`: `f` is only executed in ONE of the workers of the pool. So ONE of the processes in the pool will run `f(args)`.
>
>>`Pool.map(f, iterable)`: This method chops the iterable into a number of chunks which it submits to the process pool as separate tasks. So you take advantage of all the processes in the pool.
>
> Both of them blocks until the complete result is returned. If you want the Pool of worker processes to perform many function calls asynchronously: `Pool.apply_async()` and `Pool.map_async()`.

`map()` calls for a list of jobs in one time, thus it has to be an iterable.

``` python 
results = Pool.map(func, [1, 2, 3])
```

`apply()` on th contrary can only be called for one job, and the output has the same format with `func`.

``` python 
for x, y in [[1, 1], [2, 2]]:
    results.append(Pool.apply(func, (x, y)))
```

Everytime we call `__getitem__` we expect one single sample with a pair of data(image and bbox), and it is better -->

Finally, return the `TensorImage` and corresponding `bbox` (keypoints coordinates) and get our dataset sample.

### c. Convert Image Metas

> **When automatic batching is enabled**, `collate_fn` is called with a list of data samples at each time. It is expected to collate the input samples into a batch for yielding from the data loader iterator. The rest of this section describes the behavior of the default collate_fn (default_collate()).
>
> For instance, if each data sample consists of a 3-channel image and an integral class label, i.e., each element of the dataset returns a tuple `(image, class_index)`, the default `collate_fn` collates a list of such tuples into a single tuple of a batched image tensor and a batched class label Tensor. In particular, the default `collate_fn` has the following properties:
>
> + It always prepends a new dimension as the batch dimension.
>
> + It automatically converts NumPy arrays and Python numerical values into PyTorch Tensors.
>
> + It preserves the data structure, e.g., if each sample is a dictionary, it outputs a dictionary with the same set of keys but batched Tensors as values (or lists if the values can not be converted into Tensors). Same for `list` s, `tuple` s, `namedtuple` s, etc.
>

`default_collate()` would not be enough for this work. 

**Firstly**, turning multiple dataset tuples, `(TensorImage, bbox)`, into batch and transform it into `torch.Tensor` is necessary. Taking a batch of multiple dataset samples as input:

``` python 
    batch = self.collate_fn([self.dataset[i] for i in indices])   # old DataLoader
```

Reschedule it via `zip(*)`, and formulate a tensor image batch:

``` python 
    TensorImages, bboxes = [*zip(*batch)]       # or via `list(zip(*batch))`
    TensorImgBatch = torch.concat([_.unsqueeze(0) for _ in TensorImages])
```

![Unzip Barch](/localdata/assets/ZeroProject/UnZipBatch.png){: .shadow}
_Unzip a Tuple Set_

<!-- 
``` bash
[(0, array([0.15416284, 0.7400497 , 0.26331502, 0.53373939, 0.01457496])), 
(1, array([0.77770241, 0.23754122, 0.82427853, 0.9657492 , 0.97260111])), 
(2, array([0.51394334, 0.77316505, 0.87042769, 0.00804695, 0.30973593])), 
(3, array([0.8488177 , 0.17889592, 0.05436321, 0.36153845, 0.27540093])), 
(4, array([0.22329108, 0.52316334, 0.55070146, 0.04560195, 0.36072884])), 
(5, array([0.294665  , 0.53058676, 0.19152079, 0.06790036, 0.78698546])), 
(6, array([0.65037424, 0.50545337, 0.87860147, 0.18184023, 0.85223307])), 
(7, array([0.0975336 , 0.76124972, 0.24693797, 0.13813169, 0.33144656])), 
(8, array([0.5881308 , 0.89771373, 0.89153073, 0.81583748, 0.03588959])), 
(9, array([0.04872488, 0.28910966, 0.72096635, 0.02161625, 0.20592277])), 
(10, array([0.20846054, 0.48168106, 0.42053804, 0.859182  , 0.17116155])), 
(11, array([0.51729788, 0.9469626 , 0.76545976, 0.28239584, 0.22104536]))]
``` -->

**Secondly**, we need conduct some additional process on bbox information, convert it into metas that includes text query for decode. As mentioned before, every plot is formed by a sequence of keypoints with a variable length. Calculate the maximum of this length within a batch:

``` python 
    max_seq_length = max([b.shape[0] for b in bboxes]) + 2      # 2 for additional maskers on start "<SOS>" and end "<EOS>" of a sequence.
```

Pad with zero on empty blocks, and output an mask on this data:

``` python 
    b_padded = np.zeros((max_seq_length, 4))    # 4 for (x, y, w, h) and it is zero on w and h
    b_padded[1:b.shape[0]+1] = b

    b_mask = np.zeros(max_seq_length)
    b_mask[1:b.shape[0]+1] = 1
```

Again, here the `bbox` is nothing similar to an actual bounding box, it's just for convenience and `w` and `h` are all ignored. We are just trying to solve the problem as an Object Detection and based on query-based methods. And the meta information for every plot sample was like: 

``` python 
    img_metas.append(
        {
            ...

            'text': ','.join(["<SOS>"]+["<thead>"]*b.shape[0]+["<EOS>"]+["<PAD>"]*(max_seq_length-b.shape[0]-2)),
            'bbox': b_padded, 
            'bbox_masks': b_mask
        }
    )
```

Finally, we got all we need: random synthetic plots `TensorImgBatch`, each of them is made up by a sequence of keypoints, and corresponding meta informations `img_metas` for decoder which consist of ground-truth, query text and mask. That's the whole objective for this post. 
