# Endoscopy NeRF Tutorial --- IMR-Summer-School(Inspired from REIM-NeRF)

## Baseline: Reproduce REIM-NeRF (MICCAI 2023)

### Download the pre-trained models and normalized C3VD sequences

#### Using the provided script

The following script to downloads and extracts:
 - all pre-trained models used in the paper under `./ckpts/c3vd/main_results_iter15000_w270_h216/`.
 - all normalized C3VD pose sequences in the appropriate format to render C3VD data using the provided models. 
  
From the root directory of the repository, run:

```bash
python -m scripts/download_supporting_files.py
```

#### Download manually

We host additional files supporting the paper in UCL's research repository.
You can download a copy of all the pre-trained models from [here](https://rdr.ucl.ac.uk/articles/model/REIM-NeRF_pretrained_models_on_C3VD/24418297)
and all normalized C3VD pose sequences from [here]() (data are uploaded to github until UCL storage approval is granted. )

If you want to render C3VD sequences without training models nor pre-processing C3VD, download an extract the data above either using the script or manually and continue reading from [Render endoscopic sequences](#endering-c3vd-from-trained-models).

### Data pre-processing workflow

To train and test our models you need to have a copy of the registered videos of [C3VD](https://durrlab.github.io/C3VD/)
and based on it, generate the following:

- json files describing each dataset, including calibration parameters, file-paths and etc.
- distance maps ( like depth-maps but encoding distance in 3D from the camera center instead of just z distance).
- pixel masks, masking out edge pixels where calibration parameters are expected to be less accurate and therefore, depth and rgb information is best to be ignored during training.

1. Download a copy of all registered videos .zip files from C3VD and extract them under a single directory.
After downloading and extracting all datasets, your local copy directory tree should look like this:

    ```tree
    c3vd_registered_videos_raw
    ├── trans_t1_a_under_review
    │   └── ...
    ├── trans_t2_b_under_review
    │   ├── 0000_color.png
    │   ├── 0000_depth.tiff
    │   ├── 0000_normals.tiff
    │   ├── 0000_occlusion.png
    │   ├── ...
    │   ├── coverage_mesh.obj
    │   └── pose.txt
    └── ...
    ```

2. Run the provided preprocessing script which will generate the training, evaluation and test datasets

    ```bash
    python -m scripts.pre-process_c3vd\
            {path_to_c3vd_registered_videos_raw dir}\
            {path_to_a_directory_to_store_the_data}
    ```

    The rest of the parameters should remain default.

   By default the script is set to process the latest C3VD format ( as of 11/23). If you are using the under review version of C3VD, which was used to develop REIM-NeRF, append the above command with `--old_C3VD`. Additionally if you want to overwrite existing data add the `--overwrite` flag.

After both steps are complete, the process dataset should have the following structure

```tree
processed
   ├── trans_t1_a/(t1v1 for the new version or trans_t1_a_under_review)
   │   ├── images
   │   │   ├── 0000_color.png
   │   │   ├── 0001_color.png
   │   │   └── ....
   │   ├── distmaps
   │   │   ├── 0000_color.png
   │   │   ├── 0001_color.png
   │   │   └── ....
   │   ├── rgb_mask.png
   │   ├── transforms_test.json
   │   ├── transforms_train.json
   │   ├── transforms_true_test.json
   │   └── transforms_val.json
   └── ....
```

- **images/**: contain a copy of the source rgb images.
- **distmaps/**:  contain the distance maps named after the rgb image they correspond to. Note, those files are different from the depthmaps provided with the dataset and encode ray distance expressed in (0,pi) scale instead of z distance expressed in (0,100mm) scale.
- **rgb_mask.png**: is used during both training and validation to mask out pixels in the periphery of the frames where calibration parameters are expected to be less accurate.
- **transforms_test.json**: information for every frame, we use this to re-render video sequences
- **transforms_train.json**: information for training frame
- **transforms_eval.json**: information for evaluation frames
- **transforms_true_test.json**: information for frames used to generate the paper's results. This json does not contain training frames.

### Training

We provide scripts to train all variants of models presented in the paper, across all C3VD sequences. This allows readers to recreate the ablation study presented in the paper.

1. Go through all the steps of the [Data pre-processing workflow](#data pre-processing workflow) sections
2. Modify `train_nerf.sh`, `train_nerf_depth.sh`, `train_nerf_plus_light-source.sh`, `train_reim-nerf.sh` under `REIM-NeRF/scripts/bash/c3vd/training`, by replacing the placeholder value of variable `dataset_root_dir` with the path of the root directory of the pre-processed C3VD dataset (generated in step 1).
3. Modify the above scripts to match your system GPU resources.
4. Run the appropriate script:
   - `train_all.sh`: trains all models. It runs the following 4 scripts in sequence
   - `train_nerf.sh`: trains only the vanilla nerf
   - `train_nerf_depf.sh`: trains only the vanilla nerf with sparse depth supervision
   - `train_nerf_plus_light-source.sh`: trains the light-source location conditioned model.
   - `train_reim-nerf.sh`: trains the light-source conditioned model with sparse depth supervision. This is our full model

### Rendering C3VD from trained models

The provided scripts assume will work with models and trajectories downloaded using the provided scripts. If you download the models and trajectory paths using the provided python script, skip steps 1 and 2, the hardcoded paths in the scripts of step 3 should work.

1. Go through all the steps of the [Data pre-processing workflow](#data pre-processing workflow) sections. This is required because the rendering process relies on camera poses.
2. Modify `inference_nerf.sh`, `inference_nerf_depth.sh`, `inference_nerf_plus_light-source.sh`, `inference_reim-nerf.sh` under `REIM-NeRF/scripts/bash/c3vd/inference`, by replacing the placeholder value of variable `sequences_root_dir` with the path of the root directory of the pre-processed C3VD dataset (generated in step 1). Furthermore, replace the `checkpoints_root_dir` with the root directory of saved models for C3VD for each of the models.
3. Run the appropriate script:
   - `inference_all.sh`: run inference on C3VD with all models. This script runs the following 4 scripts in sequence
   - `inference_nerf.sh`: Inference with the original nerf model
   - `inference_nerf_depf.sh`: Inference with the original nerf model, trained with sparse depth supervision
   - `inference_nerf_plus_light-source.sh`: Inference with the light-source location conditioned model.
   - `inference_reim-nerf.sh`: Inference with the light-source location conditioned model, trained with sparse depth supervision. This is our full model

### Evaluate models trained on C3VD

1. Go through all the steps of the [Data pre-processing workflow](#data pre-processing workflow) sections.
2. Either train or download the pre-trained models(download script and links will be added shortly)
3. Modify `scripts/bash/c3vd/evaluate/evaluate_template.sh`, by changing the placeholders variables paths. `dataset_root_dir` should point to the pre-processed version of C3VD and `predictions_root_dir` should point to the root directory containing checkpoints for all c3vd dataset of a specific model variant.



## Bonus: Three on endoscopy NeRF: Gaussian-Splatting, Surgical instrument removal, EndoSLAM-NeRF

After completing the tasks mentioned above, we provide three interesting options for you to explore:  Reconstruction based on Gaussian-splatting, Surgical instruments removal, and SLAM-based NeRF.

### Gaussian Splatting

Gaussian splatting is a novel 3D reconstruction way which achieves state-of-the-art visual quality while maintaining competitive training times and importantly allow high-quality real-time (≥ 30 fps) novel-view synthesis at 1080p resolution.
Detail information can be found at: https://github.com/graphdeco-inria/gaussian-splatting/tree/main


### Surgical instrument removal






