# Training the CNN

These instructions cover all the steps necessary to train CNN models for use in the `cnn_demultiplexer classify` command.


## Prerequisites

Lots of barcoded reads! More training data is always better.


## Run Porechop

To prepare training data, we first need to know which reads have which barcodes. To do this, we run Porechop:
```
porechop -i "$fastq_dir" -b porechop_dir --require_two_barcodes --verbosity 3 --no_split > porechop.out
rm -r porechop_dir
```

By running Porechop with its highest verbosity (`--verbosity 3`), we produce an output with all of the information we need. CNN demultiplexer will read the Porechop output file, so we can delete the binned reads after it finishes.

The `--require_two_barcodes` option ensures that reads are only given a barcode bin if they have a good match on their start and end. This serves two purposes. First, it makes binning stringent, reducing the risk of misclassified reads. Second, allows us to use each read for training both the read start and read end CNNs, simplifying the process.

The `--no_split` option turns off Porechop's middle-adapter search and we use it here simply to save time. A chimeric read with a middle adapter is not a problem for our training set.



## Extract signal data

Use the `porechop` subcommand to process the Porechop output into a file of raw training data:
```
cnn_demultiplexer porechop porechop.out /path/to/fast5_dir > raw_training_data
```

This will produce a tab-delimited file with the following columns:
* `Read_ID`: e.g. 3d873ba9-d55a-45a4-9ad5-db4f64135f11
* `Barcode_bin`: a number from 1-12
* `Start_read_signal`: signal from the start of the read. This is variable in length, because some read signals begin with a fair amount of open pore signal which will be trimmed off in the next step.
* `Middle_read_signal`: signal from the middle of the read (used for training samples without a barcode)
* `End_read_signal`: signal from the end of the read. Like the start read signal, this is variable in length.



## Balancing the training data

This command finalises the training data:
```
cnn_demultiplexer balance raw_training_data training
```

It produces two separate files, one for read starts and one for read ends. Each file only has two columns: the barcode label and the signal. It balances the data by ensuring that each barcode has the same number of samples (necessarily limited to the number of samples for the least abundant barcode). No-barcode samples are included as well, using signal from the middle of reads and randomly generated signals.

This command also attempts to trim off open-pore signal at the start of the signal, so it outputs data from when the real read has begun. However, this process isn't perfect (more on that in the refining step below).



## Training the neural network

Now it's time to actually train the CNN!

```
cnn_demultiplexer train training_read_starts read_start_model
cnn_demultiplexer train training_read_ends read_end_model
```

This part can be quite time consuming, and so a big GPU is definitely recommended! Even with a big GPU, it may take many hours to finish.

Options to change some parameters:

* `--signal_size`
* `--barcode_count`
* `--epochs`: Too few and the network won't get good enough. Too many and it's a waste of time.
* `--batch_size`: Larger values may work better but will use more memory.
* `--test_fraction`: What fraction of the data will be set aside for use as a validation set. If you have no interest in assessing the model, set this to 0.0 to use all of your data for training.

If you would like to [design your own CNN architecture](https://keras.io/getting-started/functional-api-guide/), you'll need to code it up yourself by modifying the `ksudhfsodiufh` function in `network_architecture.py`.



## Refining the training set and retraining

The training data we prepared earlier may have some duds in it. This may be due to incorrect trimming of open pore signal. Or it may be that Porechop produced a false positive. Either way, our training data may have a signal which is supposed to have a particular barcode but does not. This is not good for training!

To remedy this, we use our trained CNN to classify our training data:

```
cnn_demultiplexer classify training_read_starts read_start_model > read_starts_classification
cnn_demultiplexer classify training_read_ends read_end_model > read_ends_classification
```

As you would expect, this process should classify most samples to the same barcode bin as their training label. The small fraction that do not match should be excluded from the training set. These commands will produce new training sets with those samples filtered out:

```
cnn_demultiplexer refine training_read_starts read_starts_classification > training_read_starts_refined
cnn_demultiplexer refine training_read_ends read_ends_classification > training_read_ends_refined
```

Now that a new, better training set is available, we can retrain our CNN and hopefully produce even better models than before.

```
rm read_start_model read_end_model
cnn_demultiplexer train training_read_starts_refined read_start_model
cnn_demultiplexer train training_read_ends_refined read_end_model
```