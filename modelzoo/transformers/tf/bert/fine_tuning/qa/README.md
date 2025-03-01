# TensorFlow BERT fine-tuning model for SQuAD v1.1

- [Model overview](#model-overview)
- [Sequence of the steps to perform](#Sequence-of-the-steps-to-perform)
- [Key features from CSoft platform used in this reference implementation](#Key-features-from-CSoft-platform-used-in-this-reference-implementation)
- [Code structure](#code-structure)
- [Dataset preparations](#dataset-preparations)
  - [Data download](#data-download)
  - [Data preprocessing](#Data-preprocessing)
- [Input function pipeline](#input-function-pipeline)
  - [Features dictionary](#Features-dictionary)
  - [Labels tensor](#Labels-tensor)
- [How to run](#How-to-run)
  - [Training on GPU and CPU](#Training-on-GPU-and-CPU)
  - [Prediction on GPU and CPU](#Prediction-on-GPU-and-CPU)
  - [Evaluation on GPU and CPU](#Evaluation-on-GPU-and-CPU)
  - [How to compile and validate](#How-to-compile-and-validate)
  - [How to run training on Cerebras System](#How-to-run-training-on-Cerebras-System)
- [Configs included for this model](#Configs-included-for-this-model)


# Model overview 
This model uses the [BERT](https://arxiv.org/abs/1810.04805) architecture to solve a question answering task.

This is an extractive task in the sense that the answer is a segment of text taken verbatim from the article itself.
Here we use the Stanford Question Answering Dataset (SQuAD) consisting of questions related to a set of Wikipedia articles and the corresponding answers.
A full description of the dataset and task is available in this link [SQuAD v1.1](https://rajpurkar.github.io/SQuAD-explorer/).


This directory contains the scripts to train the BERT model on the SQuAD v1.1 question answering (Q/A) task.
This is usually treated as a fine-tuning task, where the BERT model is initialized with weights generated by pre-training the model on a masked language modeling task (see [BERT](../../README.md)).
In this context, the goal of the model is to select the portion of the provided input context that corresponds to the answer to the question.
Accordingly, the BERT model architecture needs to be adjusted for the Q/A task being addressed here.
The masked language modeling (MLM) and next sentence prediction (NSP) output heads used in pre-training are replaced by a token classification head.
This new classification head is added to the output of the BERT encoder stack.
It head has a dense layer with two outputs. The first output predicts the location of the start
of the answer in the provided context while the second output predicts the location of the end
of the answer in the provided context. The parameters for the new output layers are initialized with random values.

Note: the PyTorch version of this model is located at [PyTorch Q/A](../../../../pytorch/bert/fine_tuning/qa).

# Sequence of the steps to perform
The following block diagram shows a high-level view of the sequence of steps you will perform in this example:
<p align = "center">
<img src = ./images/steps-tf-qa.png>
</p>
<p align = "center">
Fig.1 - Flow Chart of steps to fine-tune Q/A model.
</p>

# Key features from CSoft platform used in this reference implementation
Fine-tuning classification model configs are supported in the [Layer Pipelined mode](https://docs.cerebras.net/en/latest/cerebras-basics/cerebras-execution-modes.html#layer-pipelined-mode).

# Code structure
* `configs/`: YAML configuration files.
* `input/`: Input pipeline implementation based on the [SQuAD dataset](https://rajpurkar.github.io/SQuAD-explorer/). Vocab files are located in `transformers/vocab/`.
* `model.py`: Model implementation leveraging `BertForQuestionAnsweringModel` from `BertForQuestionAnsweringModel.py` for Q/A fine-tuning task.
* `data.py`: The entry point to the data input pipeline code.
* `run.py`: Training script. Performs training and validation.
* `utils.py`: Miscellaneous helper functions.
* `run_prediction.py`: Script to run prediction.

# Dataset preparations

## Data download

The SQuAD website no longer has links to the v1.1 data.
However, train and dev sets for SQuAD v1.1 can be downloaded by running the next commands:
```shell 
wget https://rajpurkar.github.io/SQuAD-explorer/dataset/train-v1.1.json &&
wget https://rajpurkar.github.io/SQuAD-explorer/dataset/dev-v1.1.json
```

## Data preprocessing

Once downloaded, the raw JSON formatted data is pre-preprocessed and saved to TFRecord formatted files to allow for fast data streaming to the system.
To create TFRecords, run the command:

```shell
python input/json2tfrecords.py \
    --vocab_file /path/to/vocab_file.txt \
    --input_file /path/to/train-v1.1.json \
    --output_dir /path/to/output_dir
```

Running the script with the `--help` option will output a description of the command line arguments.

We will focus only on the arguments that are already listed in this command. `--input_file` is the path to `train-v1.1.json` file generated at the [Data download](#Data-download) step.
Your vocabulary file `--vocab_file` should be the same one as used for pre-training, and can be found in the appropriate pre-training config file under `train_input.vocab_file` field.  

The command line options `--do_lower_case`, `--max_seq_length`, `--doc_stride`, and `--max_query_length` can be changed from their default values if necessary.

This script will generate a TFRecord file called `train.tf_record` which will be placed in the directory designated in the `--output_dir` command line option.

# Input function pipeline

If you want to use your own data loader with this example code, then this section describes the input data format expected by `BertQuestionAnsweringModel` class defined in [BertQuestionAnsweringModel.py](./BertQuestionAnsweringModel.py) file.
When you create your own custom BERT Q/A input function or dataloader, you must ensure that your input function produces a features dictionary and a label tensor as described in this section.


The input to the model takes the format of a question followed by a context which contains the answer.
The question is preceded by the `[CLS]` special token and is separated from the context by the `[SEP]` special token.
The BERT model pre-training uses a special segment embedding layer at the input to signify the two sentences used in the NSP task.
Here, that segment embedding is used to distinguish between the question and context segments of the input (see [below for details](#features-dictionary)).


The labels for this model indicate the index of the tokens from the context segment of the input that represent the answer to the question.
This is supplied to the model in the form of `2` integers specifying the start and end index of the input text to the model that comprises the answer to the question.


## Features dictionary

The features dictionary has the following key/values:

- `input_ids`: Input token IDs, padded with `0` to `max_sequence_length`. The tokens in the dataset are mapped to these IDs using the vocabulary file. These values should be between `0` and `vocab size - 1`, inclusive.  The first token should be the special `[CLS]`.  The question and context should be separated by the special `[SEP]` token.  Also, the end of the content should be marked by another `[SEP]` token.
 Here is an example of the input:
    ```
    [CLS] <question> [SEP] <context> [SEP]
    ```
  - Shape: `[batch_size, max_sequence_length]`.
  - Type: `tf.int32`

- `input_mask`: Mask for padded positions. Has values `1` on the padded positions and `0` elsewhere.
  - Shape: `[batch_size, max_sequence_length]`
  - Type: `tf.int32`

- `segment_ids`: Segment IDs. A tensor the same size as the `input_ids` designating to which segment each token belongs. Each element of this tensor should only take the value `0` or `1`.  The `[CLS]` token at the start, the question and the subsequent `[SEP]` token should all have the segment value `0`.  The context and the subsequent `[SEP]` token should have the segment value `1`. All padding tokens after the last `[SEP]` should be in segment id `0`.
  - Shape: `[batch_size, max_sequence_length]`
  - Type: `tf.int32`



An example of the input string and segment ID structure (note the input tokens are converted to IDs using the vocab file):

```
Input Tokens:   [CLS] How old is John [SEP] John lives in California. He is 20 years old. [SEP] [PAD] [PAD] .....
Segments:   0    0   0   0   0    0    1     1    1   1          1  1  1  1     1    1      0    0   .....
Labels: (10, 14)  # location of the start and end of the answer
```


## Labels tensor

The label tensor of shape (`batch_size`, `2`) and type `tf.int32`. The first and second column contain the start and end index of the answer, respectively.
These indices are referring to the `input_ids` tensor supplied in the feature dictionary.
All of these indices should be greater than the index of the `[SEP]` token which designated the start of the context segment of the input.


For the example above, the answer is contained in the line `He is 20 years old.`  Therefore, the labels would be the tuple `(10, 14)`


# How to run
## Training on GPU and CPU

Training this task can be started from a set of model weights pre-trained on a language modeling task or from scratch using random weights.

To train the model, run:

```shell
python run.py \
    --params /path/to/yaml \
    --model_dir /path/to/model_dir \
    --mode train \
    --checkpoint_path /path/to/pretrained/checkpoint
```

where `--checkpoint_path` can be left out if training from scratch.
The params yaml should include the path to the input data in the `train_input.train.tf_record` parameter. 
The list of provided param yaml files is located at [Configs included for this model ](#Configs-included-for-this-model) section.
If fine-tuning a pre-trained model, then it can be useful to define the model by passing in the pretraining yaml using `model.pretrain_params_path`
See [configs/params_bert_base_squad.yaml](configs/params_bert_base_squad.yaml) for an example.


## Prediction on GPU and CPU

To evaluate the model on the eval dataset, first we must generate the output predictions.


After training is complete, we can form predictions on the eval set using:

```shell
python run_prediction.py \
    --params /path/to/yaml \
    --predict_file /path/to/dev-v1.1.json \
    --checkpoint_path /path/to/fine_tuned/checkpoint \
    --output_dir /path/to/output_dir
```

where the params passed in should be the ones that were used to create the checkpoint located at `--checkpoint_path`, `--do_lower_case`, `--max_seq_length`, `--doc_stride`, and `--max_query_length` need to be adjusted if non-default values were used during the [Data preparation](#Data-preparation) step described above.
To generate more than one prediction per example, `--n_best_size` can be increased.
After running this script, the `--output_dir` will contain `predictions.json` and `nbest_predictions.json` as well as an eval TFRecord.

## Evaluation on GPU and CPU

The output of the prediction script used above conforms with the output required by the official SQuAD v1.1 evaluations script.
In order to evaluate the predictions, download [evaluate-v1.1.py](https://github.com/allenai/bi-att-flow/blob/master/squad/evaluate-v1.1.py), the official SQuAD evaluation script, and run:

```shell
python evaluate-v1.1.py /path/to/dev-v1.1.json /path/to/predictions.json
```

which will print a dictionary with keys `exact_match` and `f1`, the evaluation metrics for this task, on the terminal.

## To compile/validate, run train and eval on Cerebras System

Please follow the instructions on our [quickstart in the Developer Docs](https://docs.cerebras.net/en/latest/wsc/getting-started/cs-appliance.html).

> **Note**: To specify a BERT pretrained checkpoint use: `--checkpoint_path` is the path to the saved checkpoint from BERT pre-training,`--is_pretrained_checkpoint` flag is needed for loading the pre-trained BERT model for fine-tuning.


# Configs included for this model 
In order to train the model, you need to provide a yaml config file. Some popular yaml config files are listed below for reference. 
Also, feel free to create your own following these examples:  

* [params_bert_base_squad.yaml](configs/params_bert_base_squad.yaml) has a standard bert-base config with `hidden_size=768`, `num_hidden_layers=12`, `num_heads=12`.
* [params_bert_large_squad.yaml](configs/params_bert_large_squad.yaml) has a standard bert-large config with `hidden_size=1024`, `num_hidden_layers=24`, `num_heads=16`.
