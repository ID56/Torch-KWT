# Understanding the Config File

The config file contains all the hyperparameters and various settings regarding your training runs. I'll break down the numerous settings here as clearly as possible.

## Dataset Paths

```
data_root: ./data/                           # Where you have extracted the google speech commands v2 dataset.
val_list_file: ./data/validation_list.txt    # The speech commands dataset contains these two files, to provide the
test_list_file: ./data/testing_list.txt      # train-test-val split
```

## Experiment Settings

```
exp:
    wandb: False                           # Whether to use wandb or not
    wandb_api_key: <path/to/api/key>       # Path to your key. Ignored if wandb is False.
    proj_name: torch-kwt                   # Name of your wandb project. Ignored if wandb is False.
    exp_dir: ./runs                        # Your checkpoints will be saved locally at exp_dir/exp_name
    exp_name: exp-0                        # ..for example, ./runs/exp-0/something.pth
    device: auto                           # "auto" checks whether cuda is available; if not, uses cpu. You can also put in "cpu" or "cuda" as device.
                                           # only single   device training is supported currently.
    log_freq: 5                            # Saves logs every log_freq steps
    log_to_file: True                      # Saves logs to exp_dir/exp_name/training_logs.txt
    log_to_stdout: True                    # Prints logs to stdout
    val_freq: 1                            # Validate every val_freq epochs
    n_workers: 1                           # Number of workers for dataloader
    pin_memory: True                       # Pin memory argument for dataloader
    cache: 2                               # 0 -> no cache | 1 -> cache wav arrays | 2 -> cache MFCCs (and also prevents wav augmentations like time_shift,
                                           # resampling and add_background_noise)
```

## Hyperparameters
```
hparams:                    # everything nested under hparams are hyperparamters, and will be logged as wandb hparams as well.
    ...
    ...
```
### Basic settings
```
hparams:
    seed: 0                  # Random seed for determinism
    batch_size: 16           # Batch size
    n_epochs: 10             # How many epochs will be trained. (1 epoch = (len(dataset) / batch_size) steps)
    l_smooth: 0.1            # If a positive float, uses LabelSmoothingLoss instead of the vanilla CrossEntropyLoss
```

### Audio Processing
```
hparams:
    ...
    ...
    audio:
        sr: 16000            # sampling rate
        n_mels: 40           # number of mel bands for melspectrogram (and MFCC)
        n_fft: 480           # n_fft, window length, hop length, center are also all args for calculating the melspectrogram.
        win_length: 480      # Check the docs here for further explanation: 
        hop_length: 160      # https://librosa.org/doc/main/generated/librosa.feature.melspectrogram.html#librosa.feature.melspectrogram
        center: False        # MFCC conversion is currently done on CPU with librosa. May add in a CUDA MFCC conversion later (with nnAudio)
```

### Model Settings
```
hparams:
    ...
    ...
    model:   
        input_res: [98, 40]  # Shape of input spectrogram (T x n_mels)
        patch_res: [40, 1]   # Resolution of patches
        num_classes: 35      # Number of classes
        mlp_dim: 768         # mlp_dim, dim, heads, depth are all parameters which construct the model.
        dim: 192             # You may refer to the original paper and change these around to form KWT-1, KWT-2 and KWT-3. The settings to 
        heads: 3             # the left are for KWT-3, for example.
        depth: 12            #
        dropout: 0.0         # The paper uses a dropout of 0, but you may try increasing this.
        emb_dropout: 0.1     # The paper doesn't say anything about embedding dropout, but ViT uses 0.1 by default.
        pre_norm: False      # Prenorm or Postnorm transformer. The paper says that it uses Postnorm transformers, so that's the default.
```

### Optimizer & Scheduling
```
hparams:
    ...
    ...
    optimizer:               # AdamW with an lr of 0.001 and weight decay of 0.1, as in the original paper.
        opt_type: adamw      # Please modify get_optimizer() in utils/opt.py if you want to add support for more optimizer variants.
        opt_kwargs:
          lr: 0.001
          weight_decay: 0.1
    
    scheduler:               # Warmup scheduling for 10 epochs and cosine annealing, as in the original paper.
        n_warmup: 10         # Please modify get_scheduler() in utils/scheduler.py if you want to add support for other scheduling techniques.
        scheduler_type: cosine_annealing
```

### Augmentation
```
hparams:
    ...
    ...
    augment:                 # Augmentations are applied only during training
        resample:            # Randomly resamples between 85% and 115%
            r_min: 0.85
            r_max: 1.15
        
        time_shift:          # Randomly shifts samples left or right, up to 10%
            s_min: -0.1
            s_max: 0.1

        bg_noise:            # Adds background noise from a folder containing noise files. Make sure folder only contains .wav files
                             # (the default noise folder in the speech commands v2 dataset contains a .md file)
            bg_folder: ./data/_background_noise_/

        spec_aug:            # Spectral augmentation. SpecAug is applied on the CPU currently, with a numba JIT compiled function. May provide a 
                             # CUDA SpecAug later.
            n_time_masks: 2
            time_mask_width: 25
            n_freq_masks: 2
            freq_mask_width: 7
```