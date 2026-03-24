# SLEAP on the OSPool

[SLEAP](https://sleap.ai/) (Social LEAP Estimates Animal Poses) is an open-source deep learning framework for multi-animal pose estimation. This tutorial demonstrates how to train a SLEAP model, run inference, and evaluate results on the [OSPool](https://osg-htc.org/services/open_science_pool.html) - a national-scale distributed computing resource - using the [sleap-nn](https://nn.sleap.ai/) backend.

The example dataset consists of fruit fly (*Drosophila melanogaster*) video frames and follows the [sleap-nn Getting Started tutorial](https://nn.sleap.ai/latest/getting-started/first-model/). Included are HTCondor submit files for training, inference, and evaluation.

## Getting Started

Clone this repository on your OSPool access point:

<pre class="term"><code>$ git clone https://github.com/osg-htc/tutorial-sleap</code></pre>

Move into the new directory:

<pre class="term"><code>$ cd tutorial-sleap</code></pre>

The repository contains:

| File / Directory | Description |
|---|---|
| `config.yaml` | SLEAP-NN training configuration |
| `train.pkg.slp` | Packaged training dataset (fruit fly frames + labels) |
| `val.pkg.slp` | Packaged validation dataset |
| `training.sub` | HTCondor submit file for model training |
| `inference.sub` | HTCondor submit file for pose prediction |
| `evaluate.sub` | HTCondor submit file for model evaluation |
| `models/` | Output directory for trained model checkpoints |
| `logs/` | HTCondor job logs |
| `container/sleap.def` | Apptainer definition file to build a custom container |

### Configuration

The `config.yaml` file controls training behavior. Key settings:

| Parameter | Default | Description |
|---|---|---|
| `trainer_config.max_epochs` | `100` | Maximum training epochs |
| `model_config.backbone_config.unet.filters` | `32` | Model size (16=small, 32=medium, 64=large) |
| `data_config.preprocessing.scale` | `1.0` | Image scale factor (0.5 = half size, faster) |
| `trainer_config.train_data_loader.batch_size` | `4` | Increase if you have more GPU memory |
| `trainer_config.optimizer.lr` | `0.0001` | Learning rate |
| `trainer_config.early_stopping.patience` | `10` | Epochs without improvement before stopping |

### Software environment

The submit files use a prebuilt Apptainer image with SLEAP installed. If you are 
interested in your own custom environment, see the [Customizing for your work](#customizing-this-tutorial-for-your-own-work) section below. 

## Workflow

The typical workflow is:

1. **Train** a model on labeled data
2. **Infer** poses on new data using the trained model
3. **Evaluate** model performance

### 1. Training

The training job reads: 
- model config - `config.yaml`
- the training and validation datasets - `train.pkg.slp`, and `val.pkg.slp`

Then trains a single-instance pose estimation model, and writes the resulting model to  the sub-folder `models/fly_single_instance/`. 

Submit the training job:

<pre class="term"><code>$ condor_submit training.sub</code></pre>

Monitor the job:

<pre class="term"><code>$ condor_q</code></pre>

or

<pre class="term"><code>$ condor_watch_q</code></pre>

Logs are written to the `logs/` directory. After the job completes, the `models/` directory will contain:

```
models/fly_single_instance/
├── best.ckpt                  # Best model weights
├── initial_config.yaml        # Your original config
├── training_config.yaml       # Full config with auto-computed values
├── labels_gt.train.0.slp      # Training data split (ground truth)
├── labels_gt.val.0.slp        # Validation data split (ground truth)
├── labels_pr.train.slp        # Predictions on training data
├── labels_pr.val.slp          # Predictions on validation data
├── metrics.train.0.npz        # Training metrics
├── metrics.val.0.npz          # Validation metrics
└── training_log.csv           # Loss per epoch
```

These model outputs will be used in the next step, inference. 

> Interested in learning more about the basics of HTCondor job submission and 
> monitoring? See the OSPool guide on [Submit Jobs to the OSPool using HTCondor](https://portal.osg-htc.org/documentation/htc_workloads/workload_planning/htcondor_job_submission/) 
> or [Data Staging and Transfer to Jobs](https://portal.osg-htc.org/documentation/htc_workloads/managing_data/overview/). 

### 2. Inference

Once training is complete, submit the inference job to generate pose predictions on the validation dataset:

<pre class="term"><code>$ condor_submit inference.sub</code></pre>

This runs `sleap-nn track` against `val.pkg.slp` using the trained model outputs from the previous step and the same config file as before (`config.yaml`). The job produces an output dataset of predictions called `predictions.slp`. 

### 3. Evaluate

The final step compares the prediction dataset from step 2 (`predictions.slp`) with the original dataset (`val.pkg.slp`). 

Submit the evaluation job to measure model accuracy:

<pre class="term"><code>$ condor_submit evaluate.sub</code></pre>

This job should produce an output file called `metrics.npz`. See the [tutorial](https://nn.sleap.ai/latest/getting-started/first-model/#understanding-metrics) for details on understanding the metrics.

## Customizing this tutorial for your own work

Here are some items to consider if applying this tutorial to your own SLEAP workflow: 

* **Data** We are using pre-cleaned and formatted SLEAP datasets in this this example. If you have your own data, you will need to get it "SLEAP-ready" before going through the steps in this tutorial. 
	The data in this example is hosted on the [Open Science Data Federation](https://osg-htc.org/services/osdf.html). If you have an account on the primary [OSPool Access Points](https://portal.osg-htc.org/documentation/overview/account_setup/comanage-access/), you have your own space where you can host data this way, that is only accessible to you. For more information on uploading data to your OSDF folder, see [OSDF](https://portal.osg-htc.org/documentation/htc_workloads/managing_data/osdf/)

* **Configuration** Update the `config.yaml` file to reflect the parameters you want to use for your analysis. If you are interested in trying different sets of parameters, contact the facilitation team for an example of how to run a batch of jobs that each uses a different configuration. 

* **Software** If you prefer to build your own container, a definition file is provided at `container/sleap.def`. Build it with:
<pre class="term"><code>$ apptainer build sleap.sif container/sleap.def</code></pre>
Then update the `container_image` line in each `*.sub` file to point to your new image.

### Resources

- [SLEAP documentation](https://sleap.ai/)
- [sleap-nn documentation](https://nn.sleap.ai/)

