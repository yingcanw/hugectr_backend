# DeepFM SAMPLE 
A sample of deploying DeepFM Network with Hugectr backend [(link)](https://www.ijcai.org/Proceedings/2017/0239.pdf).

## Dataset and preprocess 
The data is provided by CriteoLabs (http://azuremlsampleexperiments.blob.core.windows.net/criteo/day_1.gz).
Each example contains a label (1 if the ad was clicked, otherwise 0) and 39 features (13 integer features and 26 categorical features).
The dataset also has the significant amounts of missing values across the feature columns, which should be preprocessed accordingly.
The original test set doesn't contain labels, so it's not used.


### Requirements
* Python >= 3.6.9
* Pandas 1.0.1
* Sklearn 0.22.1

### 1. Download the dataset and preprocess 

Go to [(link)](http://azuremlsampleexperiments.blob.core.windows.net/criteo/day_1.gz)
and download the kaggle-display dataset into the folder "${project_home}/tools/".


####  Download the Kaggle Criteo dataset using the following command: 
```shell.
$  wget wget http://azuremlsampleexperiments.blob.core.windows.net/criteo/day_1.gz
```

#### Extract the dataset using the following command:
```shell.
$ tar zxvf day_1.gz
```

#### preprocess the data using the following commands:
The script `preprocess.py` fills the missing values by mapping them to the unused unique integer or category.
It also replaces unique values which appear less than six times across the entire dataset with the unique value for missing values.
Its purpose is to reduce the vocabulary size of each column while not losing too much information.
In addition, it normalizes the integer feature values to the range [0, 1],
but it doesn't create any feature crosses. Please choose one of the following two methods for data preprocessing.
```shell.
$ mkdir deepfm_data
$ shuf train.txt > train.shuf.txt
$ python3 ./preprocess.py --src_csv_path=train.shuf.txt --dst_csv_path=deepfm_data/train.out.txt --normalize_dense=1 --feature_cross=0
```

#### Split the dataset using the following commands:
```shell.
$ head -n 36672493 deepfm_data/train.out.txt > deepfm_data/train && \\\\
$ tail -n 9168124 deepfm_data/train.out.txt > deepfm_data/valtest && \\\\
$ head -n 4584062 deepfm_data/valtest > deepfm_data/val && \\\\
$ tail -n 4584062 deepfm_data/valtest > deepfm_data/test
```
#### Convert the criteo data to inference format
The HugeCTR inference requires dense features, embedding columns and row pointers of slots as the input and gives the prediction result as the output. We need to convert the criteo data to inference format (csr) first.
```shell.
$ ./criteo2predict.py --src_csv_path=deepfm_data/test --src_config=../samples/deepfm/deepfm_data.json --dst_path=./deepfm_csr.txt --segmentation ',' --batch_size=1
```
As result, CSR format input will be generated into deepfm_csr.txt and the content as below:
```shell.
Label:0
DES: 0.005376344086021505 0.0008673026886383349 0.002331002331002331 0.004651162790697674 0.006083972751905593 0.0008793527963418925 0.007590132827324478 0.00591715976331361 0.0420323325635104 0.20000000000000004 0.021428571428571432 0.0 0.00646551724137931
CATCOLUMN: 630 1741 169492 439138 549150 549420 559916 561648 562203 595960 617230 785371 951890 954587 961209 1127998 1268021 1272637 1273122 1274952 1284808 1599234 1599246 1661028 1679074 1713689
ROWINDEX: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26
```


## 2. Get Ddeepfmeepfm trained model files
Go to [(deepfm training sample))](https://github.com/NVIDIA/HugeCTR/tree/master/samples/deepfm#training-with-hugectr) in HugeCTR and make sure store the trained dense model and embedding table files into the folder "${project_home}/samples/deepfm/1/"

## 3. Create inference configuration files
### Deepfm model network configuration 
Check the stored model files that will be used in the inference, and create the JSON file for inference. We should remove the solver and optimizer clauses and add the inference clause in the JSON file. The paths of the stored dense model and sparse model(s) should be specified at "dense_model_file" and "sparse_model_file" within the inference clause. We need to make some modifications to "data" in the layers clause. Besides, we need to change the last layer from BinaryCrossEntropyLoss to Sigmoid. The rest of "layers" should be exactly the same as that in the training JSON file. You may go to "${project_home}/samples/deepfm/1/deepfm.json" for reference.

### Hugectr backend configuration 
Please refer to  [(Triton model configuration))](https://github.com/triton-inference-server/server/blob/master/docs/model_configuration.md) first and o clarify the required configuration of the model in the specific inference scenario.
For deploy the Hugectr model, Some customized configuration items need to be added as follows：
```json.
 parameters [
  {
  key: "config"
  value: { string_value: "/model/deepfm/1/deepfm.json" }
  },
   {
  key: "gpucache"
  value: { string_value: "true" }
  },
  {
  key: "gpucacheper"
  value: { string_value: "0.5" }
  },
  {
  key: "label_dim"
  value: { string_value: "1" }
  },
  {
  key: "slots"
  value: { string_value: "26" }
  },
  {
  key: "cat_feature_num"
  value: { string_value: "26" }
  },
  {
  key: "des_feature_num"
  value: { string_value: "13" }
  },
  {
  key: "max_nnz"
  value: { string_value: "2" }
  },
  {
  key: "embedding_vector_size"
  value: { string_value: "32" }
  },
  {
  key: "embeddingkey_long_type"
  value: { string_value: "false" }
  }
]
```
## 4. Launch Triton Server to load DCN and Deepfm 
Before you can use the Hugectr Docker image you must install Docker. If you plan on using a GPU for inference you must also install the NVIDIA Container Toolkit. DGX users should follow Preparing to use NVIDIA Containers. 

Pull the image using the following command.
```shell.
$ docker pull nvcr.io/nvidia/hugectr_backend:v3.0-inference
```
In this sample, the DCN model and Deepfm model can be deployed simultaneously with multiple model instances in the same GPU.
Use the following command to run Triton with the deepfm and dcn sample model repository. The NVIDIA Container Toolkit must be installed for Docker to recognize the GPU(s). The --gpus=1 flag indicates that 1 system GPU should be made available to Triton for inferencing.  If building HugeCTR Backend from Scratch, please specify "--backend-directory" argument value as the absolute path that installs the HugeCTR backend.
```shell.
 docker run --gpus=2 --rm  -p 8005:8000 -p 8004:8001 -p 8003:8002  -v /hugectr_backend/sampes/:/model  nvcr.io/nvidia/hugectr_backend:v3.0-inference  tritonserver --model-repository=/model/ --backend-directory=/usr/local/hugectr/backends/ \
--backend-config=hugectr,dcn=/model/dcn/1/dcn.json --backend-config=hugectr,deepfm=/model/deepfm/1/deepfm.json  \
```
All the models should show "READY" status to indicate that they loaded correctly. If a model fails to load the status will report the failure and a reason for the failure. If your model is not displayed in the table check the path to the model repository and your CUDA drivers.
```shell.
+---------+---------------------------------------------+---------------------------------------------+
| Backend | Config                                      | Path                                        |
+---------+---------------------------------------------+---------------------------------------------+
| hugectr | /hugectr_backend/hugectr/libtriton_hugectr. | {"cmdline":{"dcn":"/model/dcn/1/dcn.json",  |
|         | so                                          | "deepfm":"/model/deepfm/1/deepfm.json"}}    |
+---------+---------------------------------------------+---------------------------------------------+

I0127 13:55:52.421274 119 server.cc:184]
+----------------+---------+--------+
| Model          | Version | Status |
+----------------+---------+--------+
| deepfm         | 1       | READY  |
| dcn            | 1       | READY  |
+----------------+---------+--------+

...
...
...
I1002 21:58:57.891440 62 grpc_server.cc:3914] Started GRPCInferenceService at 0.0.0.0:8001
I1002 21:58:57.893177 62 http_server.cc:2717] Started HTTPService at 0.0.0.0:8000
I1002 21:58:57.935518 62 http_server.cc:2736] Started Metrics Service at 0.0.0.0:8002
```

Use Triton’s ready endpoint to verify that the server and the models are ready for inference. From the host system use curl to access the HTTP endpoint that indicates server status.
```shell.
$ curl -v localhost:8005/v2/health/ready
...
< HTTP/1.1 200 OK
< Content-Length: 0
< Content-Type: text/plain
```
## 4. Running Deepfm Client 
Use docker pull to get the client libraries and examples image from NGC.
```shell.
$ docker pull nvcr.io/nvidia/tritonserver:<xx.yy>-py3-sdk
```
Where <xx.yy> is the version that you want to pull.For stability considerations, we recommend using 20.10. Hugectr backend provided a client example for your reference, The input data is generated in `1.Download the dataset and preprocess` part
```shell.
$ docker run --rm --net=host -v /hugectr_backend/samples/deepfm:/deepfm nvcr.io/nvidia/tritonserver:20.10-py3-clientsdk python3 /deepfm/deepfm_client.py
```
To send a request for the deepfm model. In this case we ask for the 10 samples for prediction.
```shell.
{'id': '1', 'model_name': 'deepfm', 'model_version': '1', 'parameters': {'NumSample': 1, 'DeviceID': 0}, 'outputs': [{'name': 'OUTPUT0', 'datatype': 'FP32', 'shape': [1], 'parameters': {'binary_data_size': 3328}}]}
[0.30833802]
```

