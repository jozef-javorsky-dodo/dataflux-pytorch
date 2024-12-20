# Checkpoint Demo for PyTorch Lightning

The code in this folder provides a training demo for checkpointing with PyTorch Lightning. This demo is under development.

## Limitations

* The demo currently only runs with [`state_dict_type="full"`](https://lightning.ai/docs/pytorch/stable/common/checkpointing_expert.html#save-a-distributed-checkpoint) when using FSDP.
* `requirements.txt` includes gcsfs because even though it is not used for checkpointing, PyTorch Lightning's default logger also writes to the root directory where checkpoints are saved.
* Note that saving or restoring checkpoint files will stage the checkpoint file in CPU memory during save/restore, requiring additional available CPU memory equal to the size of the checkpoint file.

## Running locally

1. Set the environment variables required to run the demo. These include:
   * `PROJECT`: The GCP project you are using
   * `CKPT_DIR_PATH`: The full path of the directory in which to save checkpoints, in the format `gs://<bucket>/<directory>/`
1. Set the optional environment variables, if desired:
   * `NUM_LAYERS`: The number of layers in the model, which affects the size of the model and therefore the size of the checkpoints
   * `ACCELERATOR`: Set to `gpu` if running on a GPU, or `cpu` if running on a CPU (default)
   * If running on a GPU, you also must set `PJRT_DEVICE` to `CUDA`. 
   * `TRAIN_STRATEGY`: Set to `fsdp` to use the FSDP strategy. The default is `ddp`. If using FSDP, you must use GPUs
   * `MIN_EPOCHS`: Minimum epochs for which training should run. Defaults to 4. For detailed explaination of min_epochs see [here](https://lightning.ai/docs/pytorch/stable/common/trainer.html#min-epochs).
   * `MAX_EPOCHS` : Maximum epochs for which training should run. Defaults to 5. For detailed explanation of max_epochs see [here](https://lightning.ai/docs/pytorch/stable/common/trainer.html#max-epochs).
   * `MAX_STEPS`: Maximum number of steps for which training can run. Defaults to 3. For more infomration on max_steps see [here](https://lightning.ai/docs/pytorch/stable/common/trainer.html#max-steps).
   * `ASYNC_CHECKPOINT`: Set to any non-empty value to use [AsyncCheckpointIO](https://lightning.ai/docs/pytorch/stable/api/lightning.pytorch.plugins.io.AsyncCheckpointIO.html#asynccheckpointio) for saving checkpoint data without blocking training. See the warning below.
1. Install requirements: `pip install -r demo/lightning/checkpoint/requirements.txt`; `pip install .`
1. Run the binary: `python3 -m demo.lightning.checkpoint.singlenode.train`

#### Using AsyncCheckpointIO

GPU utilization is optimized when using [AsyncCheckpointIO](https://lightning.ai/docs/pytorch/stable/api/lightning.pytorch.plugins.io.AsyncCheckpointIO.html#asynccheckpointio) by making non-blocking `save_checkpoint` calls during training. The image below shows an example training run demonstrating the difference AsyncCheckpointIO can make when large checkpoint saves would cause blocking network calls while uploading large checkpoint files. By using AsyncCheckpointIO, `save_checkpoint` calls are executed by a Threadpool and do not block training while the large checkpoint files are being uploaded. 

![image](https://github.com/user-attachments/assets/094f9dc5-cd79-438d-bae7-202d420b8f62)

To leverage the benefits of [AsyncCheckpointIO](https://lightning.ai/docs/pytorch/stable/api/lightning.pytorch.plugins.io.AsyncCheckpointIO.html#asynccheckpointio) in your code, create your trainer checkpoint plugin using the following Dataflux library

```python
from dataflux_pytorch.lightning import DatafluxLightningAsyncCheckpoint

dataflux_ckpt = DatafluxLightningAsyncCheckpoint(project_name=<PROJECT NAME>)
```

> [!Warning]
> According to the documentation, [AsyncCheckpointIO](https://lightning.ai/docs/pytorch/stable/api/lightning.pytorch.plugins.io.AsyncCheckpointIO.html#asynccheckpointio) is an experimental feature and currently does not verify if the save has succeeded.

## Running on GKE

These instructions assume you have an existing GKE cluster with Kueue and Jobset installed. These are installed by default if you create the cluster using [xpk](https://github.com/google/xpk).

### Build and push the Docker container

```
docker build -t my-container .
docker tag my-container gcr.io/<PROJECT_NAME>/my-container
docker push gcr.io/<PROJECT_NAME>/my-container
```

Make sure to update the container name in the yaml config file to match the one you're using.

### Run the workload on GKE

1. Connect to your GKE cluster: `gcloud container clusters get-credentials <CLUSTER_NAME> --region=<COMPUTE_REGION>`
2. Make a copy of `demo/lightning/checkpoint/singlenode/example-deploy.yaml` and update the placeholders and environment variables as needed
3. Run `kubectl apply -f <path-to-your-yaml-file>`
