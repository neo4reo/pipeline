![PipelineAI Logo](http://pipeline.ai/assets/img/logo/pipelineai-split-black-258x62.png)

## Pull PipelineAI [Sample Models](https://github.com/PipelineAI/models)
```
git clone https://github.com/PipelineAI/models
```
**Change into the new `models/` directory**
```
cd models
```

# Install PipelineAI
## System Requirements
* 8GB
* 4 Cores

## Requirements
* Install [Docker](https://www.docker.com/community-edition#/download)
* Install Python 2 or 3 ([Conda](https://conda.io/docs/install/quick.html) is Preferred)
* Install (Windows Only) Install [PowerShell](https://github.com/PowerShell/PowerShell/tree/master/docs/installation) 

## Install [PipelineAI CLI](../README.md#install-pipelinecli)
* Click [**HERE**](../README.md#install-pipelinecli) to install the PipelineAI CLI

# Train and Deploy Models
## TensorFlow
* [Train a TensorFlow Model](#train-a-tensorflow-model)
* [Deploy a TensorFlow Model](#deploy-a-tensorflow-model)

## Scikit-Learn 
* [Train a Scikit-Learn Model](#train-a-scikit-learn-model)
* [Deploy a Scikit-Learn Model](#deploy-a-scikit-learn-model)

## PyTorch
* [Train a PyTorch Model](#train-a-pytorch-model)
* [Deploy a PyTorch Model](#deploy-a-pytorch-model)

# Train a TensorFlow Model
## Inspect Model Directory
```
ls -l ./tensorflow/mnist-v3/model

### EXPECTED OUTPUT ###
...
pipeline_conda_environment.yml     <-- Required. Sets up the conda environment
pipeline_condarc                   <-- Required, but Empty is OK. Configure Conda proxy servers (.condarc)
pipeline_setup.sh                  <-- Required, but Empty is OK.  Init script performed upon Docker build
pipeline_train.py                  <-- Required. `main()` is required. Pass args with `--train-args`
...
```

## Build Training Server

Arguments between `[` `]` are optional 
```
pipeline train-server-build --model-name=mnist --model-tag=v3 --model-type=tensorflow --model-path=./tensorflow/mnist-v3/model
```
Notes:  
* If you change the model (`pipeline_train.py`), you'll need to re-run `pipeline train-server-build ...`
* `--model-path` must be relative to the current ./models directory (cloned from https://github.com/PipelineAI/models)
* Add `--http-proxy=...` and `--https-proxy=...` if you see `CondaHTTPError: HTTP 000 CONNECTION FAILED for url`
* For GPU-based models, make sure you specify `--model-chip=gpu`
* If you have issues, see the comprehensive [**Troubleshooting**](/docs/troubleshooting/README.md) section below.

## Start Training Server

```
pipeline train-server-start --model-name=mnist --model-tag=v3 --input-host-path=./tensorflow/mnist-v3/input/ --output-host-path=./tensorflow/mnist-v3/model/pipeline_tfserving/ --runs-host-path=./tensorflow/mnist-v3/model/pipeline_tfserving/ --output-host-path=./tensorflow/mnist-v3/model/pipeline_tfserving/ --train-args="--train-epochs=2 --batch-size=100"
```

Notes:
* Ignore the following warning: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.`
* If you change the model (`pipeline_train.py`), you'll need to re-run `pipeline train-server-build ...`
* `--input-host-path` and `--output-host-path` are host paths (outside the Docker container) mapped inside the Docker container as `/opt/ml/input` (PIPELINE_INPUT_PATH) and `/opt/ml/output` (PIPELINE_OUTPUT_PATH) respectively.
* PIPELINE_INPUT_PATH and PIPELINE_OUTPUT_PATH are environment variables accesible by your model inside the Docker container. 
* PIPELINE_INPUT_PATH and PIPELINE_OUTPUT_PATH are hard-coded to `/opt/ml/input` and `/opt/ml/output`, respectively, inside the Docker conatiner .
* `--input-host-path` and `--output-host-path` should be absolute paths that are valid on the HOST Kubernetes Node
* Avoid relative paths for * `--input-host-path` and `--output-host-path` unless you're sure the same path exists on the Kubernetes Node 
* If you use `~` and `.` and other relative path specifiers, note that `--input-host-path` and `--output-host-path` will be expanded to the absolute path of the filesystem where this command is run - this is likely not the same filesystem path as the Kubernetes Node!
* `--input-host-path` and `--output-host-path` are available outside of the Docker container as Docker volumes
* `--train-args` is used to specify `--train-files` and `--eval-files` and other arguments used inside your model 
* Inside the model, you should use PIPELINE_INPUT_PATH (`/opt/ml/input`) as the base path for the subpaths defined in `--train-files` and `--eval-files`
* We automatically mount `https://github.com/PipelineAI/models` as `/root/samples/models` for your convenience
* You can use our samples by setting `--input-host-path` to anything (ignore it, basically) and using an absolute path for `--train-files`, `--eval-files`, and other args referenced by your model
* You can specify S3 buckets/paths in your `--train-args`, but the host Kubernetes Node needs to have the proper EC2 IAM Instance Profile needed to access the S3 bucket/path
* Otherwise, you can specify ACCESS_KEY_ID and SECRET_ACCESS_KEY in your model code (not recommended_
* `--train-files` and `--eval-files` can be relative to PIPELINE_INPUT_PATH (`/opt/ml/input`), but remember that PIPELINE_INPUT_PATH is mapped to PIPELINE_HOST_INPUT_PATH which must exist on the Kubernetes Node where this container is placed (anywhere)
* `--train-files` and `--eval-files` are used by the model, itself
* You can pass any parameter into `--train-args` to be used by the model (`pipeline_train.py`)
* `--train-args` is a single argument passed into the `pipeline_train.py`
* Models, logs, and event are written to `--output-host-path` (or a subdirectory within it).  These paths are available outside of the Docker container.
* To prevent overwriting the output of a previous run, you should either 1) change the `--output-host-path` between calls or 2) create a new unique subfolder within `--output-host-path` in your `pipeline_train.py` (ie. timestamp).
* Make sure you use a consistent `--output-host-path` across nodes.  If you use timestamp, for example, the nodes in your distributed training cluster will not write to the same path.  You will see weird ABORT errors from TensorFlow.
* On Windows, be sure to use the forward slash `\` for `--input-host-path` and `--output-host-path` (not the args inside of `--train-args`).
* If you see `port is already allocated` or `already in use by container`, you already have a container running.  List and remove any conflicting containers.  For example, `docker ps` and/or `docker rm -f train-mnist-v3`.
* For GPU-based models, make sure you specify `--start-cmd=nvidia-docker` - and make sure you have `nvidia-docker` installed!
* For GPU-based models, make sure you specify `--model-chip=gpu`
* If you're having trouble, see our [Troubleshooting](/docs/troubleshooting) Guide.

(_We are working on making this more intuitive._)

## View Training Logs
```
pipeline train-server-logs --model-name=mnist --model-tag=v3
```

_Press `Ctrl-C` to exit out of the logs._

## View Trained Model Output (Locally)
_Make sure you pressed `Ctrl-C` to exit out of the logs._
```
ls -l ./tensorflow/mnist-v3/output/

### EXPECTED OUTPUT ###
...
1511367633/  <-- Sub-directories of training output
1511367765/
...
```
_Multiple training runs will produce multiple subdirectories - each with a different timestamp._

## View Training UI (including TensorBoard for TensorFlow Models)
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.
* This usually happens when using Docker Quick Terminal on Windows 7.
```
http://localhost:6006
```
![PipelineAI TensorBoard UI 0](http://pipeline.ai/assets/img/pipelineai-train-census-tensorboard-0.png)

![PipelineAI TensorBoard UI 1](http://pipeline.ai/assets/img/pipelineai-train-census-tensorboard-1.png)

![PipelineAI TensorBoard UI 2](http://pipeline.ai/assets/img/pipelineai-train-census-tensorboard-2.png)

![PipelineAI TensorBoard UI 3](http://pipeline.ai/assets/img/pipelineai-train-census-tensorboard-3.png)

## Stop Training Server
```
pipeline train-server-stop --model-name=mnist --model-tag=v3
```

# Deploy a TensorFlow Model

## Inspect Model Directory
_Note:  This is relative to where you cloned the `models` repo [above](#clone-the-pipelineai-predict-repo)._
```
ls -l ./tensorflow/mnist-v3/model

### EXPECTED OUTPUT ###
...
pipeline_conda_environment.yml     <-- Required. Sets up the conda environment
pipeline_condarc                   <-- Required, but Empty is OK.  Configure Conda proxy servers (.condarc)
pipeline_modelserver.properties    <-- Required, but Empty is OK.  Configure timeouts and fallbacks
pipeline_predict.py                <-- Required. `predict(request: bytes) -> bytes` is required
pipeline_setup.sh                  <-- Required, but Empty is OK.  Init script performed upon Docker build
pipeline_tfserving.config          <-- Required by TensorFlow Serving. Custom request-batch sizes, etc.
pipeline_tfserving/                <-- Required by TensorFlow Serving. Contains the TF SavedModel
...
```
Inspect TensorFlow Serving Model 
```
ls -l ./tensorflow/mnist-v3/pipeline_tfserving/

### EXPECTED OUTPUT ###
...
1510612525/  
1510612528/                <-- TensorFlow Serving finds the latest (highest) version 
...
```

## Build the Model into a Runnable Docker Image
* This command bundles the TensorFlow runtime with the model.

```
pipeline predict-server-build --model-name=mnist --model-tag=v3 --model-type=tensorflow --model-path=./tensorflow/mnist-v3/model
```
Notes:
* `--model-path` must be relative.
* Add `--http-proxy=...` and `--https-proxy=...` if you see `CondaHTTPError: HTTP 000 CONNECTION FAILED for url`
* If you have issues, see the comprehensive [**Troubleshooting**](docs/troubleshooting/README.md) section below.
* `--model-type`: **tensorflow**, **scikit**, **python**, **keras**, **spark**, **java**, **xgboost**, **pmml**, **caffe**
* `--model-runtime`: **jvm** (default for `--model-type==java|spark|xgboost|pmml`, **tfserving** (default for `--model-type==tensorflow`), **python** (default for `--model-type==scikit|python|keras`), **cpp** (default for `--model-type=caffe`), **tensorrt** (only for Nvidia GPUs)
* `--model-chip`: **cpu** (default), **gpu**, **tpu**
* For GPU-based models, make sure you specify `--model-chip=gpu`

## Start the Model Server
```
pipeline predict-server-start --model-name=mnist --model-tag=v3 --memory-limit=2G
```
Notes:
* Ignore the following warning: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.`
* If you see `port is already allocated` or `already in use by container`, you already have a container running.  List and remove any conflicting containers.  For example, `docker ps` and/or `docker rm -f train-mnist-v3`.
* You can change the port(s) by specifying the following: `--predict-port=8081`, `--prometheus-port=9091`, `--grafana-port=3001`.  
* If you change the ports, be sure to change the ports in the examples below to match your new ports.
* Also, your nginx and prometheus configs will need to be adjusted.
* In other words, try not to change the ports!
* For GPU-based models, make sure you specify `--start-cmd=nvidia-docker` - and make sure you have `nvidia-docker` installed!
* If you're having trouble, see our [Troubleshooting](/docs/troubleshooting) Guide.

## Inspect `pipeline_predict.py`
_Note:  Only the `predict()` method is required.  Everything else is optional._
```
cat ./tensorflow/mnist-v3/model/pipeline_predict.py

### EXPECTED OUTPUT ###
import os
import logging
from pipeline_model import TensorFlowServingModel             <-- Optional.  Wraps TensorFlow Serving
from pipeline_monitor import prometheus_monitor as monitor    <-- Optional.  Monitor runtime metrics
from pipeline_logger import log                               <-- Optional.  Log to console, file, kafka

...

__all__ = ['predict']                                         <-- Optional.  Being a good Python citizen.

...

def _initialize_upon_import() -> TensorFlowServingModel:      <-- Optional.  Called once at server startup
    return TensorFlowServingModel(host='localhost',           <-- Optional.  Wraps TensorFlow Serving
                                  port=9000,
                                  model_name=os.environ['PIPELINE_MODEL_NAME'],
                                  timeout=100)                <-- Optional.  TensorFlow Serving timeout

_model = _initialize_upon_import()                            <-- Optional.  Called once upon server startup

_labels = {'model_runtime': os.environ['PIPELINE_MODEL_RUNTIME'],  <-- Optional.  Tag metrics
           'model_type': os.environ['PIPELINE_MODEL_TYPE'],   
           'model_name': os.environ['PIPELINE_MODEL_NAME'],
           'model_tag': os.environ['PIPELINE_MODEL_TAG'],
           'model_chip': os.environ['PIPELINE_MODEL_CHIP']}

_logger = logging.getLogger('predict-logger')                 <-- Optional.  Standard Python logging

@log(labels=_labels, logger=_logger)                          <-- Optional.  Sample and compare predictions
def predict(request: bytes) -> bytes:                         <-- Required.  Called on every prediction

    with monitor(labels=_labels, name="transform_request"):   <-- Optional.  Expose fine-grained metrics
        transformed_request = _transform_request(request)     <-- Optional.  Transform input (json) into TensorFlow (tensor)

    with monitor(labels=_labels, name="predict"):
        predictions = _model.predict(transformed_request)       <-- Optional.  Calls _model.predict()

    with monitor(labels=_labels, name="transform_response"):
        transformed_response = _transform_response(predictions) <-- Optional.  Transform TensorFlow (tensor) into output (json)

    return transformed_response                                 <-- Required.  Returns the predicted value(s)
...
```

## Monitor Runtime Logs
* Wait for the model runtime to settle...
```
pipeline predict-server-logs --model-name=mnist --model-tag=v3

### EXPECTED OUTPUT ###
...
2017-10-10 03:56:00.695  INFO 121 --- [     run-main-0] i.p.predict.jvm.PredictionServiceMain$   : Started PredictionServiceMain. in 7.566 seconds (JVM running for 20.739)
[debug] 	Thread run-main-0 exited.
[debug] Waiting for thread container-0 to terminate.
```
Notes:
* You need to `Ctrl-C` out of the log viewing before proceeding.

## Predict in Any Language
* Use the REST API to POST a JSON document representing a number.
```
http://localhost:8080
```
![PipelineAI REST API](http://pipeline.ai/assets/img/api-embed-har-localhost.png)

![MNIST 8](http://pipeline.ai/assets/img/mnist-8-100x95.png)
```
curl -X POST -H "Content-Type: application/json" \
  -d '{"image": [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.05098039656877518, 0.529411792755127, 0.3960784673690796, 0.572549045085907, 0.572549045085907, 0.847058892250061, 0.8156863451004028, 0.9960784912109375, 1.0, 1.0, 0.9960784912109375, 0.5960784554481506, 0.027450982481241226, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.7882353663444519, 0.11764706671237946, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.988235354423523, 0.7921569347381592, 0.9450981020927429, 0.545098066329956, 0.21568629145622253, 0.3450980484485626, 0.45098042488098145, 0.125490203499794, 0.125490203499794, 0.03921568766236305, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.803921639919281, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6352941393852234, 0.9921569228172302, 0.803921639919281, 0.24705883860588074, 0.3490196168422699, 0.6509804129600525, 0.32156863808631897, 0.32156863808631897, 0.1098039299249649, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.007843137718737125, 0.7529412508010864, 0.9921569228172302, 0.9725490808486938, 0.9686275124549866, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.8274510502815247, 0.29019609093666077, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.2549019753932953, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.847058892250061, 0.027450982481241226, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.5921568870544434, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.7333333492279053, 0.44705885648727417, 0.23137256503105164, 0.23137256503105164, 0.4784314036369324, 0.9921569228172302, 0.9921569228172302, 0.03921568766236305, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.5568627715110779, 0.9568628072738647, 0.7098039388656616, 0.08235294371843338, 0.019607843831181526, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.43137258291244507, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.15294118225574493, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.1882353127002716, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6705882549285889, 0.9921569228172302, 0.9921569228172302, 0.12156863510608673, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.2392157018184662, 0.9647059440612793, 0.9921569228172302, 0.6274510025978088, 0.003921568859368563, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08235294371843338, 0.44705885648727417, 0.16470588743686676, 0.0, 0.0, 0.2549019753932953, 0.9294118285179138, 0.9921569228172302, 0.9333333969116211, 0.27450981736183167, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.4941176772117615, 0.9529412388801575, 0.0, 0.0, 0.5803921818733215, 0.9333333969116211, 0.9921569228172302, 0.9921569228172302, 0.4078431725502014, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.7411764860153198, 0.9764706492424011, 0.5529412031173706, 0.8784314393997192, 0.9921569228172302, 0.9921569228172302, 0.9490196704864502, 0.43529415130615234, 0.007843137718737125, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6235294342041016, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9764706492424011, 0.6274510025978088, 0.1882353127002716, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.18431372940540314, 0.5882353186607361, 0.729411780834198, 0.5686274766921997, 0.3529411852359772, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]}' \
  http://localhost:8080 \
  -w "\n\n"

### Expected Output ###
('{"variant": "mnist-v3-tensorflow-tfserving-cpu", "outputs":{"classes": [8], '
 '"probabilities": [[0.0013824915513396263, 0.00036483019357547164, '
 '0.003705816576257348, 0.010749378241598606, 0.0015819378895685077, '
 '6.45182590233162e-05, 0.00010775036207633093, 0.00010466964886290953, '
 '0.9819338917732239, 4.713038833870087e-06]]}}')
 
### FORMATTED OUTPUT ###
Digit  Confidence
=====  ==========
0      0.00138249155133962
1      0.00036483019357547
2      0.00370581657625734
3      0.01074937824159860
4      0.00158193788956850
5      0.00006451825902331
6      0.00010775036207633
7      0.00010466964886290
8      0.98193389177322390   <-- Prediction
9      0.00000471303883387
```
Notes:
* You may see `502 Bad Gateway` or `'{"results":["Fallback!"]}'` if you predict too quickly.  Let the server settle a bit - and try again.
* You will likely see `Fallback!` on the first successful invocation.  This is GOOD!  This means your timeouts are working.  Check out the `PIPELINE_MODEL_SERVER_TIMEOUT_MILLISECONDS` in `pipeline_modelserver.properties`.
* If you continue to see `Fallback!` even after a minute or two, you may need to increase the value of   `PIPELINE_MODEL_SERVER_TIMEOUT_MILLISECONDS` in `pipeline_modelserver.properties`.  (This is rare as the default is 5000 milliseconds, but it may happen.)
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7.
* If you're having trouble, see our [Troubleshooting](/docs/troubleshooting) Guide.

## Predict with CLI
* Before proceeding, make sure you hit `Ctrl-C` after viewing the logs in the previous step.
```
pipeline predict-server-test --endpoint-url=http://localhost:8080 --test-request-path=./tensorflow/mnist-v3/input/predict/test_request.json

### EXPECTED OUTPUT ###
...
('{"variant": "mnist-v3-tensorflow-tfserving-cpu", "outputs":{"classes": [8], '
 '"probabilities": [[0.0013824915513396263, 0.00036483019357547164, '
 '0.003705816576257348, 0.010749378241598606, 0.0015819378895685077, '
 '6.45182590233162e-05, 0.00010775036207633093, 0.00010466964886290953, '
 '0.9819338917732239, 4.713038833870087e-06]]}}')
...

### FORMATTED OUTPUT ###
Digit  Confidence
=====  ==========
0      0.00138249155133962
1      0.00036483019357547
2      0.00370581657625734
3      0.01074937824159860
4      0.00158193788956850
5      0.00006451825902331
6      0.00010775036207633
7      0.00010466964886290
8      0.98193389177322390   <-- Prediction
9      0.00000471303883387
```

## View Prediction Server Logs
* If you have any issues, you can review the logs as follows:
```
pipeline predict-server-logs --model-name=mnist --model-tag=v3
```

## Perform 100 Predictions in Parallel (Mini Load Test)
```
pipeline predict-server-test --endpoint-url=http://localhost:8080 --test-request-path=./tensorflow/mnist-v3/input/predict/test_request.json --test-request-concurrency=100
```
Notes:
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7.

## Monitor Real-Time Prediction Metrics
* Re-run the Prediction REST API while watching the following dashboard URL:
```
http://localhost:8080/dashboard/monitor/monitor.html?streams=%5B%7B%22name%22%3A%22%22%2C%22stream%22%3A%22http%3A%2F%2Flocalhost%3A8080%2Fdashboard.stream%22%2C%22auth%22%3A%22%22%2C%22delay%22%3A%22%22%7D%5D
```
Notes:
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7.

![Real-Time Throughput and Response Time](http://pipeline.ai/assets/img/hystrix-mini.png)

# Monitor Model Prediction Metrics
* Re-run the Prediction REST API while watching the following detailed metrics dashboard URL.
```
http://localhost:3000/
```
Notes:
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7.

![Prediction Dashboard](http://pipeline.ai/assets/img/request-metrics-breakdown.png)

![Dashboard Setup](http://pipeline.ai/assets/img/grafana-2-prometheus-datasource.png)

_Set `Type` to `Prometheus`._

_Set `Url` to `http://localhost:9090`._

(_Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7._)

_Create additional PipelineAI metric widgets using [THIS](https://prometheus.io/docs/practices/histograms/#count-and-sum-of-observations) guide to the Prometheus Syntax._

# Stop Model Server
```
pipeline predict-server-stop --model-name=mnist --model-tag=v3
```

# Train a Scikit-Learn Model
## Inspect Model Directory
```
ls -l ./scikit/linear/model

### EXPECTED OUTPUT ###
...
pipeline_conda_environment.yml     <-- Required. Sets up the conda environment
pipeline_condarc                   <-- Required, but Empty is OK. Configure Conda proxy servers (.condarc)
pipeline_setup.sh                  <-- Required, but Empty is OK.  Init script performed upon Docker build
pipeline_train.py                  <-- Required. `main()` is required. Pass args with `--train-args`
...
```

## View Training Code
```
cat ./scikit/linear/model/pipeline_train.py
```

## Build Training Server
```
pipeline train-server-build --model-name=linear --model-tag=v1 --model-type=scikit --model-path=./scikit/linear/model 
```
Notes:  
* `--model-path` must be relative.  
* Add `--http-proxy=...` and `--https-proxy=...` if you see `CondaHTTPError: HTTP 000 CONNECTION FAILED for url`
* For GPU-based models, make sure you specify `--model-chip=gpu`

* If you have issues, see the comprehensive [**Troubleshooting**](/docs/troubleshooting/README.md) section below.

## Start Training Server
```
pipeline train-server-start --model-name=linear --model-tag=v1 --output-host-path=./scikit/linear/model
```
Notes:
* Ignore the following warning: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.`
* For GPU-based models, make sure you specify `--start-cmd=nvidia-docker` - and make sure you have `nvidia-docker` installed!

## View the Training Logs
```
pipeline train-server-logs --model-name=linear --model-tag=v1

### EXPECTED OUTPUT ###

Pickled model to "/opt/ml/output/model.pkl"   <-- This docker-internal path maps to --output-host-path above
```

_Press `Ctrl-C` to exit out of the logs._

## View Trained Model Output (Locally)
* Make sure you pressed `Ctrl-C` to exit out of the logs.
```
ls -l ./scikit/linear/model/

### EXPECTED OUTPUT ###
...
model.pkl   <-- Pickled Model File
...
```

# Deploy a Scikit-Learn Model
## View Predictin Code
```
cat ./scikit/linear/model/pipeline_predict.py
```

## Build the Scikit-Learn Model Server
```
pipeline predict-server-build --model-name=linear --model-tag=v1 --model-type=scikit --model-path=./scikit/linear/model/
```
* For GPU-based models, make sure you specify `--model-chip=gpu`

## Start the Model Server
```
pipeline predict-server-start --model-name=linear --model-tag=v1
```
* Ignore the following warning: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.`
* For GPU-based models, make sure you specify `--start-cmd=nvidia-docker` - and make sure you have `nvidia-docker` installed!

## View the Model Server Logs
```
pipeline predict-server-logs --model-name=linear --model-tag=v1
```

## Predict with the Model 
### Curl Predict
```
curl -X POST -H "Content-Type: application/json" \
  -d '{"feature0": 0.03807590643342410180}' \
  http://localhost:8080  \
  -w "\n\n"

### Expected Output ###

{"variant": "linear-v1-scikit-python-cpu", "outputs":[188.6431188435]}
```
Notes:
* Ignore the following warning: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.`
* You may see `502 Bad Gateway` or `'{"results":["Fallback!"]}'` if you predict too quickly.  Let the server settle a bit - and try again.
* You will likely see `Fallback!` on the first successful invocation.  This is GOOD!  This means your timeouts are working.  Check out the `PIPELINE_MODEL_SERVER_TIMEOUT_MILLISECONDS` in `pipeline_modelserver.properties`.
* If you continue to see `Fallback!` even after a minute or two, you may need to increase the value of   `PIPELINE_MODEL_SERVER_TIMEOUT_MILLISECONDS` in `pipeline_modelserver.properties`.  (This is rare as the default is 5000 milliseconds, but it may happen.)
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7.
* If you're having trouble, see our [Troubleshooting](/docs/troubleshooting) Guide.

### PipelineCLI Predict
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline predict-server-test --endpoint-url=http://localhost:8080/invocations --test-request-path=./scikit/linear/input/predict/test_request.json

### EXPECTED OUTPUT ###

'{"variant": "linear-v1-scikit-python-cpu", "outputs":[188.6431188435]}'
```

# Train a PyTorch Model
## Inspect Model Directory
```
ls -l ./pytorch/mnist-v1/model

### EXPECTED OUTPUT ###
...
pipeline_conda_environment.yml     <-- Required. Sets up the conda environment
pipeline_condarc                   <-- Required, but Empty is OK. Configure Conda proxy servers (.condarc)
pipeline_setup.sh                  <-- Required, but Empty is OK.  Init script performed upon Docker build
pipeline_train.py                  <-- Required. `main()` is required. Pass args with `--train-args`
...
```

## View Training Code
```
cat ./pytorch/mnist-v1/model/pipeline_train.py
```

## Build Training Server
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline train-server-build --model-name=mnist --model-tag=v1 --model-type=pytorch --model-path=./pytorch/mnist-v1/model
```
Notes:
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
* `--model-path` must be relative.  
* Add `--http-proxy=...` and `--https-proxy=...` if you see `CondaHTTPError: HTTP 000 CONNECTION FAILED for url`
* For GPU-based models, make sure you specify `--model-chip=gpu` - and make sure you have `nvidia-docker` installed!
* If you have issues, see the comprehensive [**Troubleshooting**](/docs/troubleshooting/README.md) section below.

## Start Training Server
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline train-server-start --model-name=mnist --model-tag=v1 --output-host-path=./pytorch/mnist-v1/model
```
* Ignore the following warning: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.`
* For GPU-based models, make sure you specify `--start-cmd=nvidia-docker` - and make sure you have `nvidia-docker` installed!

## View the Training Logs
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline train-server-logs --model-name=linear --model-tag=v1

### EXPECTED OUTPUT ###

Pickled model to "/opt/ml/output/model.pth"   <-- This docker-internal path maps to --output-host-path above
```

_Press `Ctrl-C` to exit out of the logs._

## View Trained Model Output (Locally)
_Make sure you pressed `Ctrl-C` to exit out of the logs._
```
ls -l ./pytorch/mnist-v1/model/

### EXPECTED OUTPUT ###
...
model.pth   <-- Trained Model File
...
```

# Deploy a PyTorch Model
## View Prediction Code
```
cat ./pytorch/mnist-v1/model/pipeline_predict.py
```

## Build the PyTorch Model Server
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline predict-server-build --model-name=mnist --model-tag=v1 --model-type=pytorch --model-path=./pytorch/mnist-v1/model/ 
```
* For GPU-based models, make sure you specify `--model-chip=gpu` - and make sure you have `nvidia-docker` installed!

## Start the PyTorch Model Server
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline predict-server-start --model-name=mnist --model-tag=v1
```
* Ignore the following warning: `WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.`
* For GPU-based models, make sure you specify `--start-cmd=nvidia-docker` - and make sure you have `nvidia-docker` installed!

## View the PyTorch Model Server Logs
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline predict-server-logs --model-name=mnist --model-tag=v1
```

## Model Predict
### Predict with REST API
```
curl -X POST -H "Content-Type: application/json" \
  -d '{"image": [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.05098039656877518, 0.529411792755127, 0.3960784673690796, 0.572549045085907, 0.572549045085907, 0.847058892250061, 0.8156863451004028, 0.9960784912109375, 1.0, 1.0, 0.9960784912109375, 0.5960784554481506, 0.027450982481241226, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.7882353663444519, 0.11764706671237946, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.988235354423523, 0.7921569347381592, 0.9450981020927429, 0.545098066329956, 0.21568629145622253, 0.3450980484485626, 0.45098042488098145, 0.125490203499794, 0.125490203499794, 0.03921568766236305, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.32156863808631897, 0.9921569228172302, 0.803921639919281, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6352941393852234, 0.9921569228172302, 0.803921639919281, 0.24705883860588074, 0.3490196168422699, 0.6509804129600525, 0.32156863808631897, 0.32156863808631897, 0.1098039299249649, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.007843137718737125, 0.7529412508010864, 0.9921569228172302, 0.9725490808486938, 0.9686275124549866, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.8274510502815247, 0.29019609093666077, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.2549019753932953, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.847058892250061, 0.027450982481241226, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.5921568870544434, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.7333333492279053, 0.44705885648727417, 0.23137256503105164, 0.23137256503105164, 0.4784314036369324, 0.9921569228172302, 0.9921569228172302, 0.03921568766236305, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.5568627715110779, 0.9568628072738647, 0.7098039388656616, 0.08235294371843338, 0.019607843831181526, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.43137258291244507, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.15294118225574493, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08627451211214066, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.1882353127002716, 0.9921569228172302, 0.9921569228172302, 0.46666669845581055, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6705882549285889, 0.9921569228172302, 0.9921569228172302, 0.12156863510608673, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.2392157018184662, 0.9647059440612793, 0.9921569228172302, 0.6274510025978088, 0.003921568859368563, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.08235294371843338, 0.44705885648727417, 0.16470588743686676, 0.0, 0.0, 0.2549019753932953, 0.9294118285179138, 0.9921569228172302, 0.9333333969116211, 0.27450981736183167, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.4941176772117615, 0.9529412388801575, 0.0, 0.0, 0.5803921818733215, 0.9333333969116211, 0.9921569228172302, 0.9921569228172302, 0.4078431725502014, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.7411764860153198, 0.9764706492424011, 0.5529412031173706, 0.8784314393997192, 0.9921569228172302, 0.9921569228172302, 0.9490196704864502, 0.43529415130615234, 0.007843137718737125, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.6235294342041016, 0.9921569228172302, 0.9921569228172302, 0.9921569228172302, 0.9764706492424011, 0.6274510025978088, 0.1882353127002716, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.18431372940540314, 0.5882353186607361, 0.729411780834198, 0.5686274766921997, 0.3529411852359772, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]}' \
  http://localhost:8080 \
  -w "\n\n"
  
### EXPECTED OUTPUT ###

'{"variant": "mnist-v1-pytorch-python-cpu", ...}'
```
Notes:
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
* You may see `502 Bad Gateway` or `'{"results":["Fallback!"]}'` if you predict too quickly.  Let the server settle a bit - and try again.
* You will likely see `Fallback!` on the first successful invocation.  This is GOOD!  This means your timeouts are working.  Check out the `PIPELINE_MODEL_SERVER_TIMEOUT_MILLISECONDS` in `pipeline_modelserver.properties`.
* If you continue to see `Fallback!` even after a minute or two, you may need to increase the value of   `PIPELINE_MODEL_SERVER_TIMEOUT_MILLISECONDS` in `pipeline_modelserver.properties`.  (This is rare as the default is 5000 milliseconds, but it may happen.)
* Instead of `localhost`, you may need to use `192.168.99.100` or another IP/Host that maps to your local Docker host.  This usually happens when using Docker Quick Terminal on Windows 7.
* If you're having trouble, see our [Troubleshooting](/docs/troubleshooting) Guide.


### Predict with CLI
* Install [PipelineAI CLI](../README.md#install-pipelinecli)
```
pipeline predict-server-test --endpoint-url=http://localhost:8080/invocations --test-request-path=./pytorch/mnist-v1/input/predict/test_request.json

### EXPECTED OUTPUT ###

'{"variant": "mnist-v1-pytorch-python-cpu", ...}'
```
