# Object Relation Transformer

This is a PyTorch implementation of the Object Relation Transformer published in NeurIPS 2019. You can find the ArXiv version of the paper [here](https://arxiv.org/abs/1906.05963). This repository is largely based on code from Ruotian Luo's Transformer Captioning GitHub repo, which can be found [here](https://github.com/ruotianluo/Transformer_Captioning). 

The primary additions are as follows:
* Relation transformer model
* Create reports for runs on MSCOCO


## Requirements
* Python 2.7 (because there is no [coco-caption](https://github.com/tylin/coco-caption) version for python 3)
* PyTorch 0.4+ (along with torchvision)
* cider (already added as a submodule)

## License 

Object Relation Transformer is released under XXXX License (refer to LICENSE file for details).


## Data Preparation

### Download ResNet101 weights for feature extraction

Download the weights for ResNet101 (the file `resnet101.pth`) from [here](https://drive.google.com/drive/folders/0B7fNdx_jAqhtbVYzOURMdDNHSGM). Copy the weights to a folder `imagenet_weights` within the data folder:

```
mkdir data/imagenet_weights
cp /path/to/resenet/weights/resnet101.pth data/imagenet_weights
```

### Download and preprocess the COCO captions

Download the [preprocessed COCO captions](http://cs.stanford.edu/people/karpathy/deepimagesent/caption_datasets.zip) from Karpathy's homepage. Extract `dataset_coco.json` from the zip file and copy it in to `data/`. This file provides preprocessed captions and also standard train-val-test splits.

Then run:

```
$ python scripts/prepro_labels.py --input_json data/dataset_coco.json --output_json data/cocotalk.json --output_h5 data/cocotalk
```
`prepro_labels.py` will map all words that occur <= 5 times to a special `UNK` token, and create a vocabulary for all the remaining words. The image information and vocabulary are dumped into `data/cocotalk.json` and discretized caption data are dumped into `data/cocotalk_label.h5`.

Next run:
```
$ python scripts/prepro_ngrams.py --input_json ../dataset_coco.json --dict_json data/cocotalk.json --output_pkl data/coco-train --split train
```

This will preprocess the dataset and get the cache for calculating cider score.


### Download the COCO dataset and pre-extract the image features [KAB - IS THIS NECESSARY FOR OUR PAPER???]

Download the [COCO images](http://mscoco.org/dataset/#download) from the MSCOCO website. We need 2014 training images and 2014 validation images. You should put the `train2014/` and `val2014/` folders in the same directory, denoted as `$IMAGE_ROOT`.

Then run:

```
$ python scripts/prepro_feats.py --input_json data/dataset_coco.json --output_dir data/cocotalk --images_root $IMAGE_ROOT
```

`prepro_feats.py` extracts the resnet101 features (both fc feature and last conv feature) of each image. The features are saved in `data/cocotalk_fc` and `data/cocotalk_att`, and resulting files are about 200GB.

(Check the prepro scripts for more options, like other resnet models or other attention sizes.)

**Warning**: the prepro script will fail with the default MSCOCO data because one of their images is corrupted. See [this issue](https://github.com/karpathy/neuraltalk2/issues/4) for the fix, it involves manually replacing one image in the dataset.


### Download the Bottom-up features 

Download the pre-extracted features from [here](https://github.com/peteanderson80/bottom-up-attention). For the paper, the adaptive features were used.

Do the following:
```
mkdir data/bu_data; cd data/bu_data
wget https://storage.googleapis.com/bottom-up-attention/trainval.zip
unzip trainval.zip

```

Then run:
```
python script/make_bu_data.py --output_dir data/cocobu
```

This will create `data/cocobu_fc`, `data/cocobu_att` and `data/cocobu_box`. 


## Model Training and Evaluation

### Standard cross-entropy loss training

```bash
$ python train.py --id fc --caption_model fc --input_json data/cocotalk.json --input_fc_dir data/cocotalk_fc --input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 --batch_size 10 --learning_rate 5e-4 --learning_rate_decay_start 0 --scheduled_sampling_start 0 --checkpoint_path log_fc --save_checkpoint_every 6000 --val_images_use 5000 --max_epochs 30
```

The train script will dump checkpoints into the folder specified by `--checkpoint_path` (default = `save/`). We only save the best-performing checkpoint on validation and the latest checkpoint to save disk space.

To resume training, you can specify `--start_from` option to be the path saving `infos.pkl` and `model.pth` (usually you could just set `--start_from` and `--checkpoint_path` to be the same).

If you have tensorflow, the loss histories are automatically dumped into `--checkpoint_path`, and can be visualized using tensorboard.

The current command uses scheduled sampling. You can also set scheduled_sampling_start to -1 to disable it.

If you'd like to evaluate BLEU/METEOR/CIDEr scores during training in addition to validation cross entropy loss, use `--language_eval 1` option, but don't forget to download the [coco-caption code](https://github.com/tylin/coco-caption) into `coco-caption` directory.

For more options, see `opts.py`. 

The above training script should achieve a CIDEr-D score of about 1.15 [KAB VERIFY THIS NUMBER].


### Self-critical RL training

After training using cross-entropy loss, additional self-critical training produces signficant gains in CIDEr-D score.


First, copy the model from the pretrained model using cross entropy. (It's not mandatory to copy the model, just for back-up)
```
$ bash scripts/copy_model.sh fc fc_rl
```

Then
```bash
$ python train.py --id fc_rl --caption_model fc --input_json data/cocotalk.json --input_fc_dir data/cocotalk_fc --input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 --batch_size 10 --learning_rate 5e-5 --start_from log_fc_rl --checkpoint_path log_fc_rl --save_checkpoint_every 6000 --language_eval 1 --val_images_use 5000 --self_critical_after 30
```

The above training script should achieve a CIDEr-D score of about 1.28.


### Evaluate on Karpathy's test split

```bash
$ python eval.py --dump_images 0 --num_images 5000 --model model.pth --infos_path infos.pkl --language_eval 1 --beam_size 5 --split test
```


## Visualization

### Visualize caption predictions
Place all your images of interest into a folder, e.g. `images`, and run
the eval script:

```bash
$ python eval.py --model model.pth --infos_path infos.pkl --image_folder images --num_images 10
```

This tells the `eval` script to run up to 10 images from the given folder. If you have a big GPU you can speed up the evaluation by increasing `batch_size`. Use `--num_images -1` to process all images. The eval script will create an `vis.json` file inside the `vis` folder, which can then be visualized with the provided HTML interface:

```bash
$ cd vis
$ python -m SimpleHTTPServer
```

Now visit `localhost:8000` in your browser and you should see your predicted captions.

### Generate reports from runs on MSCOCO

The [create_report.py](create_report.py) script can be used in order to generate HTML reports containing results from different runs. Please see the script for specific usage examples.

The script takes as input one or more pickle files containing results from runs on the MSCOCO dataset. It reads in the pickle files and creates a set of HTML files with tables and graphs generated from the different captioning evaluation metrics, as well as the generated image captions and corresponding metrics for individual images.

If more than one pickle file with results is provided as input, the script will also generate a report containing a comparison between the metrics generated by each pair of methods.


## Results

The following are results from the paper that should be obtained by running the respective commands in `neurips_training_runs.sh`. As learning rate scheduling was not fully optimized, these values should only serve as a reference/expectation rather than what can be achieved with additional tuning.





## Citation

If you find this repo useful, please consider citing (no obligation at all):

```
@article{herdade2019image,
  title={Image Captioning: Transforming Objects into Words},
  author={Herdade, Simao and Kappeler, Armin and Boakye, Kofi and Soares, Joao},
  journal={arXiv preprint arXiv:1906.05963},
  year={2019}
}
```

Of course, please cite the original paper of models you are using (You can find references in the model files).

## Acknowledgments

Thanks to [Ruotian Luo](https://github.com/ruotianluo) for the original code. 