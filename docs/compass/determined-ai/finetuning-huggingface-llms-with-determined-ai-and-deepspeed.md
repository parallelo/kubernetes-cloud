---
description: >-
  Learn how to train or finetune a large language model from HuggingFace on
  CoreWeave Cloud
---

# Finetuning HuggingFace LLMs with Determined AI and DeepSpeed

[Hugging Face](https://huggingface.co/) is home to many open source datasets and machine learning models, along with Python libraries that allow quick and easy use of these powerful models.

DeepSpeed is an [open source](https://en.wikipedia.org/wiki/Open\_source) [deep learning](https://en.wikipedia.org/wiki/Deep\_learning) optimization library for [PyTorch](https://en.wikipedia.org/wiki/PyTorch) optimized for low latency, high throughput training, and is designed to reduce compute power and memory use for the purpose of training large distributed models.

In the following example,[ DeterminedAI](https://www.determined.ai/) will be deployed on CoreWeave Cloud, and leveraged to perform distributed training of the OPT-125m model.

## Source Code

The code referenced throughout the rest of this tutorial can be found under the `examples/deepspeed/huggingface` folder in the `coreweave/determined_coreweave` repository.

Make sure to use the `coreweave` branch:

{% embed url="https://github.com/coreweave/coreweave_determined/blob/coreweave/examples/deepspeed/huggingface/README.md" %}
Demo source code
{% endembed %}

## Cloud Resources

{% hint style="info" %}
**Note**

This guide makes several assumptions:\
\
• You have [set up the CoreWeave Kubernetes environment](../../../coreweave-kubernetes/getting-started.md).\
• You have some experience launching and using [DeterminedAI on CoreWeave Cloud](https://www.determined.ai). (If you have not done so already, it is recommended to deploy DeterminedAI via the application Catalog to familiarize yourself with it).\
• You have `git` installed on your terminal.
{% endhint %}

### Create a Shared Filesystem storage volume

First, create a **Shared Filesystem storage volume** from the **Storage** menu in the [CoreWeave Cloud UI](https://cloud.coreweave.com/storage?\_gl=1\*11x43xa\*\_ga\*MTY1MjQxMzI1Mi4xNjY4NDQwMTM1\*\_ga\_XKNHS53VYL\*MTY3MTAzMzkwMi41NC4xLjE2NzEwMzU2NjEuMC4wLjA.). This volume will be used to store the model and training data for finetuning. Shared storage volumes may be accessed by many nodes at once in CoreWeave, allowing for a massive amount of compute power to access the same dataset.

If you're following along with this demo, you can use the values shown and described below.

<figure><img src="../../.gitbook/assets/image (14).png" alt=""><figcaption><p>Create a New Volume on the Storage menu from the Cloud UI</p></figcaption></figure>

The values used for this demo are as follows:

| Field Name       | Demo Value          |
| ---------------- | ------------------- |
| **Volume Name**  | `finetune-opt-125m` |
| **Region**       | LAS1                |
| **Disk Class**   | HDD                 |
| **Storage Type** | Shared Filesystem   |
| **Size (Gi)**    | 1,000               |

{% hint style="info" %}
**Note**

If needed, it is easy to [increase the size](https://docs.coreweave.com/coreweave-kubernetes/storage#resizing) of a storage volume later.
{% endhint %}

### **Install the Filebrowser application**

The **filebrowser** application, available through the [application Catalog](https://apps.coreweave.com/), allows you to access your storage volumes via a Web interface that allows you to upload and download files and folders.

It is recommended that the name you give this filebrowser application be very short, or you will run into SSL CNAME issues. We recommend `finetune`.

Simply select the `finetune-opt-125m` PVC that you created earlier. **Make sure that you actually add your PVC to the filebrowser list of mounts!**

![The filebrowser application in the Cloud UI application Catalog](<../../.gitbook/assets/Screen Shot 2022-07-26 at 4.10.34 PM.png>)

{% hint style="info" %}
**Note**\
Installing the filebrowser application is **very helpful** to this process. As an alternative, it may be preferable to you to launch a Virtual Server or Kubernetes Pod to interact with their PVC via SSH or other mechanism.
{% endhint %}

## Install Determined AI

[Follow the steps to install the Determined AI application](install-determined-ai.md). The values used for the configuration of Determined AI in this tutorial are as follows:

#### Region

`LAS1 (Las Vegas)`

#### Default Resources

| Field                 | Demo Value |
| --------------------- | ---------- |
| **Default resources** | 8 vCPUs    |
| **Memory request**    | 256Gi      |
| **GPU Type**          | A40        |

#### Object Storage Configuration

| Field           | Demo Value                                                                             |
| --------------- | -------------------------------------------------------------------------------------- |
| **Bucket Name** | `<YOUR_BUCKET_NAME>` - this should be replaced by your actual Object Storage bucket    |
| **Access Key**  | `<YOUR_ACCESS_KEY>` - this should be replaced by your actual Object Storage access key |
| **Secret Key**  | `<YOUR_SECRET_KEY>` - this should be replaced by your actual Object Storage secret key |

#### Attaching the volume

Click `+` to attach the `finetune-opt-125m` volume.

<figure><img src="../../.gitbook/assets/image (28) (1).png" alt=""><figcaption><p>The attachment configuration screen for the DeterminedAI application</p></figcaption></figure>

As shown above, for this tutorial we are attaching the `finetune-opt-125m` volume on the mount path `/mnt/finetune-opt-125m`.

{% hint style="warning" %}
**Important**

The default username is `admin` and there is no password. Make sure to add a password after logging into the application for the first time.
{% endhint %}

## Prepare the Dataset

HuggingFace has [many datasets available](https://huggingface.co/datasets) for training machine learning models. Since this example is finetuning OPT-125m, a language model, it will use the [wikitext dataset](https://huggingface.co/datasets/wikitext). This dataset contains text extracted from the set of verified Good and Feature articles on Wikipedia.

{% hint style="info" %}
**Note**

The rest of the tutorial will reference files in the `examples/deepspeed/huggingface` folder in the [`coreweave/determined_coreweave`](https://github.com/coreweave/coreweave\_determined) repository.
{% endhint %}

The `prepare_lm_data.py` script will preprocess the dataset so it is ready to be used with finetuning. There are two important transformations that the script performs:

1. Tokenizes the data
2. Generate chunks of a configurable max length

### Upload the Script

For Determined AI to be able to run the preprocessing script, it needs access to it. It also needs to install the python requirements. To achieve this, upload the `prepare_lm_data.py` and `requirements.txt` files to the PVC that is mounted the Determined deployment.

An easy way to do this is by using the FileBrowser UI that was previously installed. To access the UI, you can find the link in the [CoreWeave Cloud Applications UI](https://apps.coreweave.com/).

{% hint style="warning" %}
**Important**

The original username and password for the FileBrowser UI are both`admin`.

Make sure to update these after logging in for the first time.
{% endhint %}

<figure><img src="../../.gitbook/assets/image (31) (1).png" alt=""><figcaption><p>FileBrowser UI for the mounted PVC after the required files have been uploaded.</p></figcaption></figure>

### Set up the Determined CLI

In order to run the script using Determined, first make sure that the [Determined CLI](https://docs.determined.ai/latest/interfaces/commands-and-shells.html) (`det`)is installed.

To install the `det` command, install the determined Python package. As it is listed in the `requirements.txt` file, it can be installed using:

```bash
pip install -r requirements.txt
```

Next, set the `DET_MASTER` environment variable to the cluster link; this can be obtained from the post-installation notes shown when the Determined AI application deployment finishes in the CoreWeave Cloud UI. If you need to access these notes, navigate to the **Applications** page, select the Determined AI application, and look for the line that begins `To access the WebUI, please visit:`. The link will be displayed there.

```bash
export DET_MASTER=...las1.ingress.coreweave.cloud
```

### Run the Script

The `prepare_lm_data.py` script accepts a number of different arguments. To see them all, run `python prepare_lm_data.py --help`.

Note that the original dataset is cached and the processed dataset is stored on the mounted PVC. This means that the data will persist there after the command is finished running and future trials can all access it simultaneously.

{% hint style="info" %}
**Note**

If you would like to use a smaller dataset to to get through this example quicker, you can change the `--dataset_config_name` value to `wikitext-2-raw-v1`.
{% endhint %}

Run the follow command to preprocess the wikitext dataset:

```bash
det cmd run 'pip install -r /mnt/finetune-opt-125m/requirements.txt; \
    python /mnt/finetune-opt-125m/prepare_lm_data.py \
    --dataset_name wikitext \
    --dataset_config_name wikitext-103-raw-v1 \
    --processed_dataset_destination /mnt/finetune-opt-125m/wikitext/processed \
    --tokenizer_name facebook/opt-125m \
    --dataset_cache_dir /mnt/finetune-opt-125m/wikitext'
```

The logs from the script will be output to your terminal window, but you can also view them from the Determined UI under the Tasks page.

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption><p>Prepare Dataset Task running in Determined AI</p></figcaption></figure>

Once the task is complete you should see the processed dataset files in the PVC via the FileBrowser.

<figure><img src="../../.gitbook/assets/image (41) (1).png" alt=""><figcaption><p>The processed WikiText dataset files shown in the FileBrowser UI.</p></figcaption></figure>

## Prepare for Training

Now that the dataset is processed, get ready to finetune OPT-125m!

### Build the Docker image

Determined AI publishes public Docker containers, which can be used when running experiments. For example, there is a Determined AI image that comes configured with PyTorch, Cuda, and DeepSpeed. However, that image does not have the `model_hub` package - so we need to make a custom image that has everything we need included.

{% hint style="warning" %}
**Important**

The default Docker tag is `latest`. Using this tag is **strongly** **discouraged**, as containers are cached on the nodes and in other parts of the CoreWeave stack. Always use a unique tag, and never push to the same tag twice. Once you have pushed to a tag, **do not** push to that tag again.
{% endhint %}

In the following example, simple versioning is used, in which the tag `1` denotes the first iteration of the image, and so on.

{% hint style="info" %}
**Note**

When running the following commands, be sure to replace the example `username` with your Docker Hub `username`.
{% endhint %}

To build the custom image defined in the `Dockerfile`, run the following commands:

```bash
docker login
export DOCKER_USER=<your dockerhub username>
docker build -t $DOCKER_USER/hf_deepspeed_det:1 .
docker push $DOCKER_USER/hf_deepspeed_det:1
```

{% hint style="info" %}
**Note**

This example assumes a public Docker registry. To use a private registry, an  [`imagePullSecret` ](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)must be defined.
{% endhint %}

### The DeepSpeed config

[The DeepSpeed configuration file](https://www.deepspeed.ai/docs/config-json/) used for the experiments in this example is defined by the `ds_config.json` file.

One important value to make note of is the [ZeRO Optimization](https://www.deepspeed.ai/tutorials/zero/) stage set to `1`. This means that not only will data parallelism split the batches across numerous instances of the model across different GPUs, but the optimizer will be partitioned to further increase efficiency.

In total, there are three stages, each of which adds another level of parallelization:

| **Stage 1**  | The optimizer state is partitioned across all devices |
| ------------ | ----------------------------------------------------- |
| **Stage 2**  | The gradients are partitioned across all devices      |
| **Stage 3**  | The model weights are partitioned across all devices  |

<figure><img src="../../.gitbook/assets/image (18).png" alt="Graph from the ZeRO paper showing the memory efficiency from different optimization stages."><figcaption><p>Graph from the <a href="https://arxiv.org/abs/1910.02054v3">ZeRO paper</a> showing the memory efficiency from different optimization stages.</p></figcaption></figure>

### The experiment config

Determined AI experiments can be defined in YAML files. The two included in this example are:

* `opt125m_single.yml`
* `opt125m_search.yml`

There are a number of different sections that define a Determined AI experiment. Those important to this tutorial are explained below.

{% hint style="info" %}
**Note**

For an in-depth explanation on Determined AI's experiment config, visit their [Experiment Configuration Reference page](https://docs.determined.ai/latest/reference/reference-training/experiment-config-reference.html#experiment-configuration-reference).&#x20;
{% endhint %}

#### Searcher

One big value add for Determined AI's platform is how it can manage entire experiments, not just single runs. Every experiment can have one or many trials. Each trial trains their own instance of the model and a set of hyperparameters. The `searcher` section defines how the experiment will set up and run all of its trials.

Two different kinds of searchers will be used in this tutorial: [`single`](https://docs.determined.ai/latest/reference/reference-training/experiment-config-reference.html#single) and [`adaptive_asha`](https://docs.determined.ai/latest/reference/reference-training/experiment-config-reference.html#single). Further explanations of each will be offered as the experiment is run.

#### Hyperparameters

The hyperparameters that are used in these experiments are described below.

| Variable                        | Description                                                                                                                           |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `pretrained_model_name_or_path` | Name of the Hugging Face model that will be used.                                                                                     |
| `cache_dir`                     | Path to the location where the model will be cached. **Important:** Use a PVC here, so every trial won't re-download the model        |
| `use_pretrained_weights`        | Determines whether to train the model from scratch or to use the published, pretrained weights                                        |
| `model_mode`                    | Used by `model_hub` when loading the necessary components                                                                             |
| `deepspeed_config_file`         | Name of the DeepSpeed JSON config file                                                                                                |
| `overwrite_deepspeed_args`      | Values that will override the matching values in the DeepSpeed config - useful when doing hyperparameter searches on DeepSpeed values |

All of the values used for these variables are selected based on choosing to finetune OPT-125M. However, you'll see in the experiment YAML files that in this demo, we are overriding the defualt DeepSpeed value for `train_micro_batch_size_per_gpu`. This value should be tuned based on the choice of your model, the kind of data being used, and the GPU that will be used.

`train_micro_batch_size_per_gpu` is the number of samples that will be given to each GPU during training. The full batch size can be calculated by `train_micor_batch_size_per_gpu * num_gpus`. A value set too low will underutilize the GPUs, but a value set too high will use too much memory. A value of `16` will make good use of the A40s used in this example without using too much memory.

{% hint style="info" %}
**Note**

DeterminedAI has their own [Introduction to Distributed Training page](https://docs.determined.ai/latest/training/dtrain-introduction.html) that will give you a further explanation of their system.&#x20;
{% endhint %}

## Run finetuning experiments

Everything is now ready for training! In these next steps, two finetuning experiments will be run on Determined, then inspected via the Web UI.

### Single trial experiment

The first experiment will run one trial with specific hyperparameters using a `single` searcher. This experiment is defined in the `opt125m_single.yml` file.

{% hint style="warning" %}
**Important**

Update the environment section of `opt125m_single.yml` to point to the Docker image that was just built:

`gpu: <your dockerhub username>/hf_deepspeed_det:1`
{% endhint %}

To create the experiment, run the following command:

```bash
det experiment create opt125m_single.yml .
```

### Grid search experiment

The second experiment will run multiple trials while performing a grid search over the `train_micro_batch_size_per_gpu`. It is defined in the `opt125m_search.yml` file.

{% hint style="warning" %}
**Important**

Updated the environment section of `opt125m_search.yml` to point to the Docker image that was just built:

`gpu: <your dockerhub username>/hf_deepspeed_det:1`
{% endhint %}

Instead of defining `train_micro_batch_size_per_gpu` to be a single value as in the previous experiment, it is now defined as a categorical, with four different possible values. Defining the value in this way tells the grid `searcher` to create trials with each of these possible values.

To create the experiment, run the following command:

```bash
det experiment create opt125m_search.yml .
```

{% hint style="info" %}
**Note**

This experiment searches over only one parameter, but you can easily expend this to search over many parameters at once. See Determined's [Hyperparameter](https://docs.determined.ai/latest/reference/reference-training/experiment-config-reference.html#hyperparameters) and [Searcher](https://docs.determined.ai/latest/reference/reference-training/experiment-config-reference.html#searcher) documentation for more information.
{% endhint %}

### The Determined experiment Web UI

You'll find the running experiments in the "Uncategorized" section of Determined's Web UI.

<figure><img src="../../.gitbook/assets/image (7) (3).png" alt=""><figcaption><p>Active experiments in Determined's Web UI</p></figcaption></figure>

#### Single trial overview

The trial overview shows a lot of important info in one tab. You can see a graph of the trial's performance over time, along with a chart of the raw number visible in the graph.

<figure><img src="../../.gitbook/assets/image (22).png" alt=""><figcaption><p>Overview tab for a Running Trial in the Determined Web UI</p></figcaption></figure>

#### Checkpoints

All checkpoints may be seen in the **Overview** tab, but each checkpoint is also given its own tab.

<figure><img src="../../.gitbook/assets/image (37).png" alt=""><figcaption><p>Checkpoints tab in the Determined Web UI</p></figcaption></figure>

Viewing a specific checkpoint's information will show you additional information like the file sizes and location of the checkpoint.

<figure><img src="../../.gitbook/assets/image (12) (3).png" alt=""><figcaption><p>Checkpoint Information Panel</p></figcaption></figure>

Using the location from the Web UI, you can also find and download the checkpoint via the `s3cmd`:

```bash
export BUCKET_NAME=<YOUR BUCKET NAME>

# List all checkpoints
s3cmd ls s3://$BUCKET_NAME

# Download a specific checkpoint
s3cmd get -r s3://$BUCKET_NAME/2923ac08-0f11-4391-945e-6de03e424a61/
```

#### Profiler

The profiler was enabled in both of the experiments that were created, but this is not the case by default. When the profiler is enabled, Determined will automatically track a number of system metrics.

<figure><img src="../../.gitbook/assets/image (9).png" alt=""><figcaption><p>Profiler Tab for a Running Experiment in Determined's Web UI</p></figcaption></figure>

For the grid search experiment, it is important to compare the throughput metrics, GPU free memory, and GPU utilization across the different micro batch size. Notice that the higher micro batch sizes use more of the GPUs memory getting higher utilization and samples per second.

#### Hyperparameters

The values used for the trial may be viewed in the hyperparameter tab. This tab is more important when performing hyperparameter searches and you want to know the specific values for that trial compared to the others.

<figure><img src="../../.gitbook/assets/image (10).png" alt=""><figcaption><p>Hyperparameter Tab for a Trial in Determined's Web UI</p></figcaption></figure>

#### Experiment with Multiple Trials Overview

From the overview of an experiment that is running multiple trials, you can track the progress of all trials.&#x20;

<figure><img src="../../.gitbook/assets/image (25).png" alt=""><figcaption><p>Experiment Overview with Multiple Trials</p></figcaption></figure>

#### Trials

You can see more information about each specific trial in the **Trials** tab. Each individual trial may also be inspected from here by clicking into them.

<figure><img src="../../.gitbook/assets/image (19).png" alt=""><figcaption><p>Trials Tab in the Determined Web UI</p></figcaption></figure>

{% hint style="info" %}
**Note**

The trial corresponding to the micro batch size of 16 (trial `5` in the above screenshot) finished much faster than the other trials. This is expected, because the GPUs were able to process the data in a more parallelized fashion.
{% endhint %}
