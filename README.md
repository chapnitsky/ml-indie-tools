# A collection of machine learning tools for low-resource research and experiments

[![License](http://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat)](LICENSE)
[![Docs](https://img.shields.io/badge/docs-stable-blue.svg)](https://domschl.github.io/ml-indie-tools/index.html)
[![PyPI version fury.io](https://badge.fury.io/py/ml-indie-tools.svg)](https://pypi.python.org/pypi/ml-indie-tools/)

## Description

```bash
pip install ml-indie-tools
```

This module contains of a collection of tools useable for researchers with limited access to compute-resources and
who change between laptop, Colab-instances and local workstations with a graphics card.

`env_tools` checks the current environment, and populates a number of flags that allow identification of run-time
environment and available accelerator hardware. For Colab instances, it provides tools to mount Google Drive for
persistent data- and model-storage.

The usage scenarios are:

| Env                      | Tensorflow TPU | Tensorflow GPU | Pytorch TPU | Pytorch GPU | Jax TPU | Jax GPU |
| ------------------------ | :------------: | :------------: | :---------: | :---------: | :-----: | :-----: |
| Colab                    | +              | +              | /           | +           | +       | +       |
| Workstation with Nvidia  | /              | +              | /           | +           | /       | +       |
| Apple Silicon            | /              | +              | /           | +           | /       | /       |

(`+`: supported, `/`: not supported)

`Gutenberg_Dataset` and `Text_Dataset` are NLP libraries that provide text data and can be used in conjuction
with Huggingface [Datasets](https://huggingface.co/docs/datasets/) or directly with ML libraries.

`ALU_Dataset` is a toy-dataset that allows training of integer arithmetic and logical (ALU) operations.

### env_tools

A collection of tools that allow moving machine learning projects between local hardware and colab instances.

#### Examples

Local laptop:

```python
from ml_indie_tools.env_tools import MLEnv
ml_env = MLEnv(platform='tf', accelator='fastest')
ml_env.describe()  # -> 'OS: Darwin, Python: 3.9.9 (Conda) Tensorflow: 2.7.0, GPU: METAL'
ml_env.is_gpu   # -> True
ml_env.is_tensorflow  # -> True
ml_env.gpu_type  # -> 'METAL'
```

Colab instance:

```python
# !pip install -U ml_indie_tools
from ml_indie_tools.env_tools import MLEnv
ml_env = MLEnv(platform='tf', accelerator='fastest')
print(ml_env.describe())
print(ml_env.gpu_type)
```

Output: 

```
DEBUG:MLEnv:Tensorflow version: 2.7.0
DEBUG:MLEnv:GPU available
DEBUG:MLEnv:You are on a Jupyter instance.
DEBUG:MLEnv:You are on a Colab instance.
INFO:MLEnv:OS: Linux, Python: 3.7.12, Colab Jupyter Notebook Tensorflow: 2.7.0, GPU: Tesla K80
The tensorboard extension is already loaded. To reload it, use:
  %reload_ext tensorboard
OS: Linux, Python: 3.7.12, Colab Jupyter Notebook Tensorflow: 2.7.0, GPU: Tesla K80
Tesla K80
```

#### Project paths

`ml_env.init_paths('my_project', 'my_model')` will give a list of paths that are adapted for local and colab usage

Local project:

```python
ml_env.init_paths("my_project", "my_model")  
# -> ('.', '.', './model/my_model', './data', './logs')
```

The list contains `<root-path>`, `<project-path>` (both are `.`, the current directory for local projects), `<model-path>` to save model and weights, `<data-path>` for
training data and `<log-path>` for logs.
  
Those paths (with exception of `./logs`) are moved to Google Drive for Colab instances: 

On Google Colab:

```
# INFO:MLEnv:You will now be asked to authenticate Google Drive access in order to store training data (cache) and model state.
# INFO:MLEnv:Changes will only happen within Google Drive directory `My Drive/Colab Notebooks/<project-name>`.
# DEBUG:MLEnv:Root path: /content/drive/My Drive
# Mounted at /content/drive
('/content/drive/My Drive',
 '/content/drive/My Drive/Colab Notebooks/my_project',
 '/content/drive/My Drive/Colab Notebooks/my_project/model/my_model',
 '/content/drive/My Drive/Colab Notebooks/my_project/data',
 './logs')
```
  
See the [env_tools API documentation](https://domschl.github.io/ml-indie-tools/_build/html/index.html#module-env_tools) for details.

### Gutenberg_Dataset

Gutenberg_Dataset makes books from [Project Gutenberg](https://www.gutenberg.org) available as dataset.

This module can either work with a local mirror of Project Gutenberg, or download files on demand.
Files that are downloaded are cached to prevent unnecessary load on Gutenberg's servers.

#### Working with a local mirror of Project Gutenberg

If you plan to use a lot of files (hundreds or more) from Gutenberg, a local mirror might be the best
solution. Have a look at [Project Gutenberg's notes on mirrors](https://www.gutenberg.org/help/mirroring.html).

A mirror image suitable for this project can be made with:

```bash
rsync -zarv --dry-run --prune-empty-dirs --del --include="*/" --include='*.'{txt,pdf,ALL} --exclude="*" aleph.gutenberg.org::gutenberg ./gutenberg_mirror
```

It's not mandatory to include `pdf`-files, since they are currently not used. Please review the `--dry-run` flag.

Once a mirror of at least all of Gutenberg's `*.txt` files and of index-file `GUTINDEX.ALL` has been generated, it can be used via:

```python
from ml_indie_tools.Gutenberg_Dataset import Gutenberg_Dataset
gd = Gutenberg_Dataset(root_url='./gutenberg_mirror')  # Assuming this is the file-path to the mirror image
```

#### Working without a remote mirror

```python
from ml_indie_tools.Gutenberg_Dataset import Gutenberg_Dataset
gd = Gutenberg_Dataset()  # the default Gutenberg site is used. Alternative specify a specific mirror with `root_url=http://...`.
```

#### Getting Gutenberg books

After using one of the two methods to instantiate the `gd` object:

```python
gd.load_index()  # load the index of books
```

Then get a list of books (array). Each entry is a dict with meta-data:
`search_result` is a list of dictionaries containing meta-data without the actual book-text.

```python
search_result = gd.search({'author': ['kant', 'goethe'], 'language': ['german', 'english']})
```

Insert the actual book text into the dictionaries. Note that download count is [limited](https://domschl.github.io/ml-indie-tools/_build/html/index.html#Gutenberg_Dataset.Gutenberg_Dataset.insert_book_texts) if using a remote server.

```python
search_result = gd.insert_book_texts(search_result)
# search_result entries now contain an additional field `text` with the filtered text of the book.
import pandas as pd
df = pd.DataFrame(search_result)  # Display results as Pandas DataFrame
df
```
  
See the [Gutenberg_Dataset API documentation](https://domschl.github.io/ml-indie-tools/_build/html/index.html#module-Gutenberg_Dataset) for details.

### Text_Dataset

A library for character, word, or dynamical ngram tokenization.

```python
import logging
logging.basicConfig(encoding='utf-8', level=logging.INFO)
from ml_indie_tools.Gutenberg_Dataset import Gutenberg_Dataset
from ml_indie_tools.Text_Dataset import Text_Dataset

gd=Gutenberg_Dataset()
gd.load_index()
bl=gd.search({'title': ['proleg', 'hermen'], 'language': ['english']})
bl=gd.insert_book_texts(bl)
for i in range(len(bl)):
    print(bl[i]['title'])
```
Prolegomena to the Study of Hegel's Philosophy<br>
Kant's Prolegomena<br>
The Cornish Fishermen's Watch Night and Other Stories<br>
Prolegomena to the History of Israel<br>
Legge Prolegomena<br>
```python
tl = Text_Dataset(bl)  # bl contains a list of texts (books from Gutenberg)
tl.source_highlight("If we write anything that contains parts of the sources, like: that is their motto, then a highlight will be applied.")
```
`INFO:Datasets:Loaded 5 texts`<br>
If we writ**e anything t**[4]**hat contains**[1] **parts of the s**[4]ources, like: **that is t**[1]**heir motto**[4], then a highligh**t will be a**[1]pplied.<br>
Sources: Julius Wellhausen: Prolegomena to the History of Israel[4], William Wallace and G. W. F. Hegel: Prolegomena to the Study of Hegel's Philosophy[1]
```python
test_text="That would be a valid argument if we hadn't defeated it's assumptions way before."
print(f"Text length {len(test_text)}, {test_text}")
tokenizer='ngram'
tl.init_tokenizer(tokenizer=tokenizer)
st = tl.tokenize(test_text)
print(f"Token-count: {len(st)}, {st}")
```
`Text length 81, That would be a valid argument if we hadn't defeated it's assumptions way before.
Token-count: 27, [1447, 3688, 1722, 4711, 4880, 1210, 1393, 4393, 2382, 1352, 3655, 1972, 1939, 44, 23, 3333, 1871, 4975, 2967, 2884, 2216, 2382, 3048, 1546, 4589, 2272, 30]`
```python
test2="ðƒ "+test_text
print(f"Text length {len(test2)}, {test2}")
el=tl.encode(test2)
print(f"Token-count: {len(el)}, {el}")
```
`Text length 84, ðƒ That would be a valid argument if we hadn't defeated it's assumptions way before.
Token-count: 29, ['<unk>', '<unk>', 1397, 3688, 1722, 4711, 4880, 1210, 1393, 4393, 2382, 1352, 3655, 1972, 1939, 44, 23, 3333, 1871, 4975, 2967, 2884, 2216, 2382, 3048, 1546, 4589, 2272, 30]`

See the [Text_Dataset API documentation](https://domschl.github.io/ml-indie-tools/_build/html/index.html#module-Text_Dataset) for details.
### ALU_Dataset

See the [ALU_Dataset API documentation](https://domschl.github.io/ml-indie-tools/_build/html/index.html#module-ALU_Dataset) for details.
A sample project is at [ALU_Net](https://github.com/domschl/ALU_Net)
  
### keras_custom_layers

A collection of Keras residual- and self-attention layers

See the [keras_custom_layers API documentation](https://domschl.github.io/ml-indie-tools/_build/html/index.html#module-keras_custom_layers) for details.

## External projects

Checkout the following jupyter notebook based projects for example-usage:

### Text generation
* [tensor-poet](https://github.com/domschl/tensor-poet)
* [torch-poet](https://github.com/domschl/torch-poet)
* [transformer-poet](https://github.com/domschl/transformer-poet)
* [torch-transformer-poet](https://github.com/domschl/torch-transformer-poet), using pytorch transformers from Andrej Karpathy's nanoGPT as implemented in [`ng-video-lecture`](https://github.com/karpathy/ng-video-lecture)

### Arithmetic and logic operations
* [ALU_Net](https://github.com/domschl/ALU_Net)

## History

* (2023-01-30, 0.5.3) Add `use_aliases` parameter to Folder- and Calibre datasets.
* (2023-01-27, 0.5.2) Add `alias` field to local datasets to protect
privacy of local document names.
* (2023-01-27, 0.5.0) Acquire training data from Calibre library ([`Calibre_Dataset`](https://domschl.github.io/ml-indie-tools/_build/html/index.html#Calibre_Dataset.Calibre_Dataset)), the documents must be in text format in Calibre, or get training data from a folder containing text files ([`Folder_Dataset`](https://domschl.github.io/ml-indie-tools/_build/html/index.html#module-Folder_Dataset)). Text_Dataset can now
contain texts from Gutenberg, Calibre or a folder of text files.
* (2023-01-26, 0.4.4) Add save/load tokenizer to Text_Dataset to enable reusing tokenizer data.
* (2023-01-22, 0.4.3) Add temperature parameter to generator.
* (2023-01-21, 0.4.2) Start of port of pytorch transformers from Andrej Karpathy's nanoGPT as implemented in [`ng-video-lecture`](https://github.com/karpathy/ng-video-lecture/blob/master/gpt.py). Additional tests with Apple Silicon MPS and pytorch 2.0 nightly.
* (2022-12-13, 0.4.0) The great cleanup: neither recurrence nor gated memory improved the transformer architecture, so they are removed again.
* (2022-12-11, 0.3.17) Testversion for slightly handwavy recurrent attention
* (2022-06-19, 0.3.1) get_random_item(index) that works with all tokenization strategies, get_unique_token_count() added.
* (2022-06-19, 0.3.0) Breaking change in Text_Dataset __get_item__() behavior, old API didn't fit with tokenization.
* (2022-06-19, 0.2.0) Language agnostic dynamic ngram tokenizer.
* (2022-06-07, 0.1.5) Support for pytorch nightly 1.13dev MPS, Apple Metal acceleration on Apple Silicon.
* (2022-03-27, 0.1.4) Bugfixes to Gutenberg `search` and `load_book` and `get_book`.
* (2022-03-15, 0.1.2) `env_tools.init()` no longer uses `tf.compat.v1.disable_eager_executition()` since there are rumors about old code-paths being used. Use `tf.function()` instead, or call with `env_tools.init(..., old_disable_eager=True)` which continues to use the old v1 API.
* (2022-03-12, 0.1.0) First version for external use.
* (2021-12-26, 0.0.x) First pre-alpha versions published for testing purposes, not ready for use.

