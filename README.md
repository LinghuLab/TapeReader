# Tape Reader Analysis Pipeline
https://github.com/user-attachments/assets/b7ecc5de-13df-4db2-9594-cd41f2c93e83


We present an end-to-end pipeline for large-scale analysis of CytoTape proteins through a comprehensive suite of tools to perform cell segmentation, fiber segmentation, and signal extraction.


Our pipeline endpoints are available in the `justfile` using [just](https://github.com/casey/just) and parameters are configured in the `config.yaml`. Once the fiber segmentation model is trained, the entire pipeline can be run using `just everything`.

Note: you will need to install [git-lfs](https://git-lfs.com/) to properly download this repo.

## Data format
We require `.tif` files located in `dataset_path` to be formatted in `(C, Z, Y, X)` axis-order, where the 0-th channel contains the cell imagery, 1-st channel contains CytoTape imagery, and the rest of the channels contain signals to be extracted.


## Fiber segmentation model training
Code for training the fiber segmentation model is stored in `fiber_segmentation_model`.
Pretrained model checkpoints are available at `fiber_segmentation_model/outputs/checkpoints` and we also include a streamlined setup for training on a new set of data. Our process requires that images be preprocessed through applying patch-wise CLAHE. Model training is done using [Pytorch Connectomics](https://connectomics.readthedocs.io/) library, on [jasonkena’s barcode branch](https://github.com/jasonkena/pytorch_connectomics/tree/barcode). Both commands can be run using `just everything` while in the `fiber_segmentation_model` folder.


Additionally, we provide evaluation functions to aid in development. Note that these commands must be run from the `fiberpipeline/segmentation/fiber_segmentation_model/scripts` folder. Given a model prediction and a corresponding ground truth, we can find optimal watershed thresholds for this specific pair by running:


```
python postprocess.py --task optimize_bcs --input [path_to_model _prediction] --output [path_to_ground_truth] -c ../../../../config.yaml
```


These thresholds must be placed in `config.yaml` in the root folder for usage. To just run watershed on a given model prediction, use:


```
python postprocess.py --task bcs_watershed --input [path_to_model _prediction] --output [save_path_for_output_segmentation] -c ../../../../config.yaml
```


Finally, we include tools to measure precision, recall, and [adapted Rand index](https://github.com/zudi-lin/pytorch_connectomics/blob/20ccfde6f85868351b00d7b795d4cf89a251d6be/connectomics/utils/evaluate.py#L18) between a segmentation and its corresponding ground truth. These stats can be calculated using:


`python evaluate.py --segmentation [path_to_segmentation] --ground_truth [path_to_ground_truth]`




## Cell segmentation
In `cell_segmentation.py`, we use [Segment Anything for Microscopy](https://github.com/computational-cell-analytics/micro-sam) to do automated instance segmentation, using their light-microscopy pretrained models.


## Skeletonization
Since the proteins do not branch, we are able to use PCA and spline-fitting to derive smooth centerlines in `skeletonize_fibers.py`. We designed some hand-crafted adjustments to ensure high-quality centerlines, including downsampling the z-axis to counteract z-axis-dilated fiber segmentations, and we fit only the middle 80% of voxels to prevent overfitting to endpoints of the fiber. We overshoot the predicted centerline by 20% length on each side.


## Signal extraction, normalization, and filtering
We perform trilinear interpolation on the signal channels in `extract_signals.py`. Further, we determine the optimal midpoint and rescaling factors for our fibers assuming symmetric signals, rescaling everything to a fixed number of points in `normalize_signals.py`. In `filter_all.py` we compute metrics which can be used to filter out fibers by quality, using geodesic length, the linearity of the centerline, mean brightness of the cellular segmentation channel, and the cell segmentation label.


## Plotting
We support viewing through both [neuroglancer](https://github.com/google/neuroglancer) and napari in `plot.py`. For neuroglancer, we use [cloud-volume](https://github.com/seung-lab/cloud-volume) and [igneous](https://github.com/seung-lab/igneous) to process each volume prior to serving, performing downsampling and meshing for visualization. We also support neuroglancer’s experimental 3D rendering for our image channels.
We support visualizing the skeletons in `napari` through point cloud representations.


Running `just plot` from the root directory will allow you to select the volume for visualization. After selecting their volume of choice, users will be able to select between `neuroglancer` and `napari`, and determine whether to plot all fibers/skeletons or just those filtered.


Extracted signals are plotted using [matplotlib](https://matplotlib.org/). Users are able to choose a volume and view the signals extracted from each of the input’s channels. Running `just plot_signals` from the root directory will allow you to select a volume of choice. You can then iterate through the extracted signals, pressing `q` to move onto the next channel.


For users on remote clusters, X11 forwarding/remote desktop/port forwarding will be necessary for visualizations through napari/matplotlib/neuroglancer.


https://github.com/user-attachments/assets/7af5ddc0-1f69-4aa0-b5bc-182c2ec7a74e

<img width="800" alt="0702-1-A5-CA1-40X001-final_signals" src="https://github.com/user-attachments/assets/484beb43-bb9d-47f2-aabe-5c2fb39442e6" />


## Contribution
**Tape Reader** (v1.0) was developed in 2025 by Jason K. Adhinarta, Michael Lin, and Donglai Wei (Department of Computer Science, Boston College), in collaboration with the Changyang Linghu lab (University of Michigan, Ann Arbor).
