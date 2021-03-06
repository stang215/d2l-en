# Machine Translation and Data Sets
:label:`chapter_machine_translation`

Machine translation (MT) refers to the automatic translation of a segment of text
from one language to another. Solving this problem with neural networks is often
called neural machine translation (NMT). Compared to the language model we discussed before, a major difference for MT is that the output is a sequence of words instead of a single words. The length of the output sequence could be different to the source sequence length. In the rest of this section, we will demonstrate how to pre-process a MT dataset and transform it into a set of data batches.

```{.python .input  n=1}
import sys
sys.path.insert(0, '..')

import collections
import d2l
import zipfile

from mxnet import nd
from mxnet.gluon import utils as gutils, data as gdata
```

## Read and Pre-process Data

We first download a dataset that contains a set of English sentences with the corresponding French translations. As can be seen that each line contains a English sentence with its French translation, which are separated by a `TAB`.

```{.python .input  n=18}
fname = gutils.download('http://www.manythings.org/anki/fra-eng.zip')
with zipfile.ZipFile(fname, 'r') as f:
    raw_text = f.read('fra.txt').decode("utf-8")
print(raw_text[0:95])
```

Words and punctuation marks should be separated by spaces. But this dataset has a few exceptions. We fix them by adding necessary spaces before punctuation marks, replacing non-breaking space with space. In addition, we convert all chars into lower cases.

```{.python .input  n=3}
def preprocess_raw(text):
    text = text.replace('\u202f', ' ').replace('\xa0', ' ')
    out = ''
    for i, char in enumerate(text.lower()):
        if char in (',', '!', '.') and i > 0 and text[i-1] != ' ':
            out += ' '
        out += char
    return out

text = preprocess_raw(raw_text)
print(text[0:95])
```

## Tokenization

A word or a punctuation mark is treated as a token, then a sentence is a list of tokens. We convert the text data into a set of source (English) sentences, a list of list of tokens, and a set of target (French) sentences. To simplify the later model training, we only sample the first `num_examples` sentences pairs.

```{.python .input  n=4}
num_examples = 50000
source, target = [], []
for i, line in enumerate(text.split('\n')):
    if i > num_examples:
        break
    parts = line.split('\t')
    if len(parts) == 2:
        source.append(parts[0].split(' '))
        target.append(parts[1].split(' '))

source[0:3], target[0:3]
```

We visualize the histogram of the number of tokens per sentence the following figure. As can be seen that a sentence in average contains 5 tokens, and most of them have less than 10 tokens.

```{.python .input  n=5}
d2l.set_figsize()
d2l.plt.hist([[len(l) for l in source], [len(l) for l in target]],
             label=['source', 'target'])
d2l.plt.legend(loc='upper right');
```

## Vocabulary

Now build a vocabulary for the source sentences and print its vocabulary sizes.

```{.python .input  n=10}
def build_vocab(tokens):
    tokens = [token for line in tokens for token in line]
    return d2l.Vocab(tokens, min_freq=3, use_special_tokens=True)

src_vocab = build_vocab(source)
len(src_vocab)
```

## Load Dataset

Since sentences have variable lengths, we define a `pad` function to trim or pad a sentence into a fixed length.

```{.python .input  n=11}
def pad(line, max_len, padding_token):
    if len(line) > max_len:
        return line[:max_len]
    return line + [padding_token] * (max_len - len(line))

pad(src_vocab[source[0]], 10, src_vocab.pad)
```

Now we can convert a list of sentences into an `(num_example, max_len)` index array. We also record the length of each sentence without the padding tokens, called valid length. In addition, we add the special “&lt;bos&gt;” and “&lt;eos&gt;” tokens to the target sentences so that our model will know the signals for starting and ending predicting.

```{.python .input  n=12}
def build_array(lines, vocab, max_len, is_source):
    lines = [vocab[line] for line in lines]
    if not is_source:
        lines = [[vocab.bos] + line + [vocab.eos] for line in lines]
    array = nd.array([pad(line, max_len, vocab.pad) for line in lines])
    valid_len = (array != vocab.pad).sum(axis=1)
    return array, valid_len
```

Finally, we construct data iterators to read data batches from the source and target index arrays.

```{.python .input  n=13}
def load_data_nmt(batch_size, max_len):  # This function is saved in d2l.
    src_vocab, tgt_vocab = build_vocab(source), build_vocab(target)
    src_array, src_valid_len = build_array(source, src_vocab, max_len, True)
    tgt_array, tgt_valid_len = build_array(target, tgt_vocab, max_len, False)
    train_data = gdata.ArrayDataset(
        src_array, src_valid_len, tgt_array, tgt_valid_len)
    train_iter = gdata.DataLoader(train_data, batch_size, shuffle=True)
    return src_vocab, tgt_vocab, train_iter
```

Let's read the first batch.

```{.python .input  n=14}
src_vocab, tgt_vocab, train_iter = load_data_nmt(batch_size=2, max_len=8)
for X, X_valid_len, Y, Y_valid_len, in train_iter:
    print('X =', X.astype('int32'), '\nValid lengths for X =', X_valid_len,
          '\nY =', Y.astype('int32'), '\nValid lengths for Y =', Y_valid_len)
    break
```

## Summary
