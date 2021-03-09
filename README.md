# UdS NNIA Project 2021

## Table of Contents

- [General](#General): information about this project
- [Pipeline](#Pipeline): technical information for set up
- - [Data Preprocessing](##Data-Preprocessing): POS extraction, aggregation
- - [Tokenization & Embedding](##Tokenization-&-Embedding): embedding creation
- - [Models](##Models): training

## General

This repository is mandatory for the UdS NNIA class of 2020/2021. This project is centered around POS prediction using BERT embeddings and variety of different models. The research question is simply comparing different architectures (dense, CNN, recurrent). Despite that, the scripts provide arguments to also research the effect of training data size on the performance.

# Pipeline

This project is written in Python, with the environment defined in `envirovnment.yml` (managed by Anaconda). We further assume, that all data are stored in `data/`.

## Data Preprocessing

- Concatenate all files into one, e.g. by `cat data/ontonetes-4.0/*.gold_conll > data/all.conll`
- Extract the tripet (word position, word, POS) using `src/data/extract_pos.sh data/all.conll data/all.tsv`
- For overview of the POS data (in .tsv), run `./src/data/info.py data/all.tsv > data/all.info`:

```
Tags:                   Overall:
NN       12.93%         Max sequence length:      73
IN        9.94%         Min sequence length:       2
DT        9.68%         Mean sequence length:  18.75
PRP       7.01%         Number of sequences:     309
,         6.44%
RB        5.56%
.         5.23%
VBD       5.20%
...
```

## Tokenization, Embedding and Training

This part tokenizes the words (in a sense of subword units + BERT specific indexing) with BERT Tokenizer. The input is then fed into BERT and the hidden states in the last layer are used as embeddings. Run `python3 src/embedding.py --name SPLIT --data data/all.tsv --data.out data/embedding_SPLIT.pkl` to compute the embeddings and store them. Replace split with `train, validation/dev, test` to produce a specific split. See [Appendix](#Appendix) for further usage details. This step also aggregates (averaging) embedding for multiple subwords for a token. A small number of sentences (8) were impossible to reconstruct using the simple parser and were discarded. The total number is reported by this script.

To train a model, run `python3 src/train.py MODEL"`. See [Models][#Models] for the list of models. Logs are stored using [Weights & Biases](https://wandb.ai/).

Evaluation on test can be done with `python3 src/test.py PATH_TO_MODEL`.

## Full Example

The full pipeline for logistic regression (model `dense1`) would then be:

```
cat data/ontonetes-4.0/*.gold_conll > data/all.conll
bash src/data/extract_pos.sh data/all.conll data/all.tsv
python3 src/embedding.py --name test --data data/all.tsv --data.out data/embedding_test.pkl
python3 src/embedding.py --name dev --data data/all.tsv --data.out data/embedding_dev.pkl
python3 src/embedding.py --name train --data data/all.tsv --data.out data/embedding_train.pkl

python3 src/train.py dense1 --data "data/embedding_"

python3 src/evaluate.py data/models/dense1/e050.pt
```

This section omitted multiple arguments when calling the scripts. Please see [Appendix][#Appendix] for more options and their defaults.

## Models

TODO

# Appendix

## Script usage

This section lists script usage dumps as well as a few miscellaneous notes.

```
usage: embedding.py [-h] [--data DATA] [--data-out DATA_OUT] [--name NAME] [--no-hotswap] [--seed SEED]

optional arguments:
  -h, --help           show this help message and exit
  --data DATA          Path to parsed tsv
  --data-out DATA_OUT  Where to store pickled embeddings
  --name NAME          Section of the data to use (train, validation/dev, test)
  --no-hotswap         Hotswap to CPU when GPU is out of memory
  --seed SEED          Seed to use for shuffling
```

Caution, `embedding.py --name train` is a highly memory sensitive task, requiring for save usage ~20GB of free memory (CPU and GPU in total). There is a mechanism that starts swapping sentences to CPU RAM when GPU is about to be full. To disable this, add `--no-hotswap`.

By default, this decreases the original precision (32-bit) to half the size (16-bit) floats. To disable this, add `--no-half`. 

```
usage: train.py [-h] [--epochs EPOCHS] [--batch BATCH] [--data DATA] [--train-size TRAIN_SIZE] [--dev-size DEV_SIZE] [--seed SEED] model

positional arguments:
  model                 Model to use

optional arguments:
  -h, --help            show this help message and exit
  --epochs EPOCHS       Number of epochs to use
  --batch BATCH         Batch size to use
  --data DATA           Prefix of path to embedding_{train,dev,test}.pkl
  --train-size TRAIN_SIZE
                        Number of training examples to use
  --dev-size DEV_SIZE   Number of dev examples to use
  --seed SEED           Seed to use for shuffling
```

## Source

Source code structure:

```
src/
├── data
│   ├── extract_pos.sh   # preprocessing to TSV
│   ├── info.py          # preprocessing summary
│   ├── ontonotes.py     # complex data loader
├── embedding.py         # computes and stores BERT embeddings
├── test.py              # evaluation on test data
├── train.py             # training of multiple models
├── utils.py             # misc. utilities
└── zoo                  # model zoo
    ├── __init__.py      # model factory, parameter definition
    ├── cnn.py           # CNN model
    ├── dense.py         # dense layers model
    ├── evaluatable.py   # model base class
    ├── most_common.py   # most common classifier
    └── rnn.py           # RNN/LSTM/GRU model
```

## Note on Difficulty

The biggest issue I was struggling with was the lack of overall memory (both CPU and GPU). This forced me to adjust the scripts so that they would be more memory aware and even implement the hotswapping mechanism in `embedding.py`. Eventually the embedding precision was halved.