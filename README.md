# HUBII microservice: HRV-02 pipeline test-melis

(Fast)API Pipeline for the processing of heart rate data from raw ECG signals towards HRV features.
It can be deployed as a microservice locally and is also available to use via a web interface [here](https://hubii.world/hrv-pipeline-02/), on the HUBII Platform.
(Note: Press the button "Try it out" to see the input forms!)

## Table of Contents
1. [HUBII pipeline description](#hubii-pipeline-description)
   1. [Input Data Form](#input-data-form)
   2. [Output Data Form](#output-data-form)
2. [HUBII pipeline workflow](#hubii-pipeline-workflow)
3. [Requirements](#requirements)
4. [Setup](#setup)
5. [Repo Structure](#repo-structure)
6. [Documentation](#documentation)

## HUBII pipeline description

This pipeline is designed to process ECG data from raw signals to HRV features. It is implemented as a FastAPI to be 
used inside an application. However, it is also designed to be possibly be used later on in a web application, where the 
user can upload ECG data and receive the HRV features. 

Short Overview:
* Input (plus device): HDF5 files with sensor data (e.g. Output of Recording Software
    OpenSignals Revolution (https://support.pluxbiosignals.com/knowledge-base/introducing-opensignals-revolution/) by Bioplux (https://www.pluxbiosignals.com/)) + User Input Fields
* Output: JSON response, JSON fiel, CSV response, CSV file, Excel Spreadsheet file
* Input data form:
  * JSON implementing ECGBatch and ECGSample models. See [below](#input-data-form) for more details and 
  an example is given in [the data directory](/data/examples/example1_input.json).
  * HDF5 files containing the raw ECG data and user Input fields.
* Output data form: JSON, CSV or Excel Spreadsheet implementing a Pandas DataFrame containing the HRV features for each window for each sample given in the 
batch. See [below](#output-data-form) for more details and an example is given in [the data directory](/data/examples/example1_output.json).

### Input Data Form
Examples for the Input are implemented in the code and can be seen when running the api via the /docs user interface. 
However, here is an overview of the Input model and its attributes, including their type, constraints and description.

This code defines two Pydantic input models, ECGSample and ECGBatch, for validating and processing data related to 
electrocardiogram (ECG) biosignals. 

**Note**: The nested Config class of each pydanctic model is used to set additional options for the class configuration, 
such as defining an example value for the model then seen on the actual interface. It overrides the Config class of the BaseModel.
For more information, see the [Pydantic documentation](https://pydantic-docs.helpmanual.io/usage/model_config/).

**ECGBatch** is a model for representing a batch of ECG data from multiple subjects in an experiment. It has the following attributes:

| **ECGBatch**  | **Type**                | **Optional** | **Constraints** | **Description**                                                                          |
|---------------|-------------------------|--------------|-----------------|------------------------------------------------------------------------------------------|
| supervisor    | str                     | False        | N/A             | Name of the supervisor who conducted the experiment.                                     |
| record_date   | date                    | False        | N/A             | Date of the recording. If not provided, the current date is set as the default.          |
| configs       | ECGConfig               | True         | N/A             | Additional configurations for the experiment.                                            |
| samples       | List[ECGSample]         | False        | min_items=1     | List of ECGSample objects representing the results of the experiment for all subjects.  |

**ECGSample** is a model for representing the results of a single subject in an ECG experiment. It has the following attributes:

| **ECGSample**   | **Type**            | **Optional** | **Constraints** | **Description**                                                              |
|-----------------|---------------------|--------------|-----------------|------------------------------------------------------------------------------|
| sample_id       | UUID                | True         | N/A             | Unique ID of the sample. If not provided, a UUID is generated automatically. |
| subject_id      | str                 | False        | N/A             | ID of the subject for whom the ECG data was collected.                        |
| frequency       | int                 | False        | N/A             | Sampling frequency of the ECG data.                                           |
| device_name     | str                 | True         | N/A             | Name of the device used to record the ECG data.                                |
| timestamp_idx   | List[datetime]      | False        | min_items=2     | List of timestamps at which the ECG data was recorded.                        |
| ecg             | List[float]         | False        | min_items=2     | List of ECG signal data values recorded at each timestamp.                    |
| label           | List[str]           | True         | min_items=2     | List of labels corresponding to each timestamp in the timestamp_idx list.     |

**ECGConfig** is a model for representing the configuration settings used during an ECG experiment. It has the following attributes:

| **ECGConfig**         | **Type**           | **Optional** | **Constraints** | **Description**                                                             |
|-----------------------|--------------------|--------------|-----------------|-----------------------------------------------------------------------------|
| signal                | str                | True         | N/A             | Name of the ECG signal.                                                      |
| window_slicing_method | str                | True         | N/A             | Method used to slice the ECG data into windows.                               |
| window_size           | float              | True         | N/A             | Size of the window used to calculate the features.                            |
| baseline_start        | datetime           | True         | N/A             | Start time of the baseline window.                                            |
| baseline_end          | datetime           | True         | N/A             | End time of the baseline window.                                              |
| baseline_duration     | int                | True         | N/A             | Duration of the baseline window in seconds.                                   |
| normalization_method  | str                | True         | N/A             | Method used for normalization.                                                |
| extra                 | Dict[str, Any]     | True         | N/A             | Additional configuration options.                                            |

Please note that the `ECGBatch` model represents a batch of ECG data, where each `ECGSample` corresponds to the results of a single subject in the experiment. The `ECGConfig` model contains the configuration settings used during the experiment.

These models can be used to validate and process the input and output data in your ECG feature processing application.

### Output Data Form

The output data is a Pandas DataFrame containing the HRV features for each window for each sample given in the batch. This is then converted to a simple records JSON format. The DataFrame has the following columns:

| Column        | Type     | Description                                                                                                 |
|---------------|----------|-------------------------------------------------------------------------------------------------------------|
| sample_id     | str      | ID of the sample.                                                                                           |
| subject_id    | str      | ID of the subject.                                                                                          |
| window_id     | int      | ID of the window.                                                                                           |
| w_start_time  | datetime | Start time of the window.                                                                                   |
| w_end_time    | datetime | End time of the window.                                                                                     |
| baseline      | bool     | Whether the window is a baseline window or not. There is and can be only one baseline window for each sample.|
| frequency     | int      | Sampling frequency of the ECG data.                                                                          |
| HRV_MeanNN    | float    | Mean of the RR intervals in the window.                                                                      |
| HRV_SDNN      | float    | Standard deviation of the RR intervals in the window.                                                       |
| HRV_RMSSD     | float    | Root mean square of the successive differences of the RR intervals in the window.                           |
| HRV_pNN50     | float    | Percentage of successive differences of the RR intervals in the window that are greater than 50 ms.         |
| ...           | ...      | ...                                                                                                         |
| hrv_feature_n | float    | Additional HRV feature n.                                                                                   |

Please note that in addition to the mentioned HRV features (HRV_MeanNN, HRV_SDNN, HRV_RMSSD, HRV_pNN50), there can be more HRV features included in the DataFrame.

## HUBII pipeline workflow
The pipeline workflow consists of several steps to convert, process ECG data and extract HRV features. Here is an overview of the pipeline:

### Step 1: Data Conversion and Preprocessing 🔄

The first step, data conversion and preprocessing, is responsible for transforming the input data into the necessary format and performing initial data preprocessing operations. This step accommodates various data formats including JSON, CSV, Excel, or HDF5. Its primary objective is to ensure that the input data is in an appropriate format for subsequent processing steps. During this phase, the content of the input data is mapped to the `ECGBatch` model, which serves as the foundational structure for all subsequent processing. This approach guarantees a consistent data format and enables universal processing across different input sources.
### Step 2: ECG Preprocessing 🧹

The ECG preprocessing step applies a cleaning algorithm to the raw ECG signals. It uses the `ecg_clean` function from the `neurokit2` library with the Pantompkins algorithm. The cleaned ECG signals are stored in the `ecg` column of the sample dataframe.

### Step 3: Window Creation 🪟

The window creation step cuts out windows from the preprocessed ECG signals based on the specified window size and window slicing method. The `create_windows` function is used to create windows with the specified parameters. The resulting windows are stored in a list.

### Step 4: Baseline Window Processing (Optional) ⚖️

If a baseline window is specified in the configuration, an additional step is performed to process the baseline window. The baseline window is cut out using the `cut_out_window` function and processed using the `process_window` function. The processed baseline window is appended to the features list.

### Step 5: Window Processing 🔄

The window processing step iterates over each window in the list. The `process_window` function is applied to each window to extract HRV features. The processed window features are appended to the features list.

### Step 6: Features DataFrame Concatenation 📊

After processing all the windows, the features list is concatenated into a single dataframe using `pd.concat`. This dataframe contains the HRV features for all windows.

### Step 7: Feature Normalization (Optional) 📏

If a baseline window is specified in the configuration, a feature normalization step is performed. The normalize_features function is applied to the features dataframe to normalize the features based on the specified normalization method.
* Normalization Methods:
  * Difference: Subtract the baseline features from the non-baseline features.
  * Relative: Divide the non-baseline features by the baseline features.
  * Separate or None: No normalization is applied, this is the case if either the baseline should just be given as any other window ('Separate') or even no baseline is defined ('None').

### Step 8: Output 💾

The resulting features dataframe is returned as the output of the pipeline.

---

This workflow can be tracked via logging during the execution of the endpoints in the terminal:
![terminal_output.png](app/figures/terminal_output.png)

---

The pipeline workflow provides a systematic process for preprocessing ECG data, creating windows, extracting HRV features, and optionally normalizing the features using the baseline. Each step contributes to the overall goal of analyzing ECG data and deriving meaningful HRV insights.

## Endpoints and Functionality 🛠️

The FastAPI implementation of the pipeline provides the following endpoints (available endpoint tags: 🔖[feature processing], 🔖[conversion]):

### 1. Process Features by Raw JSON Input 📥 (🔖[feature processing])

- **Endpoint**: `/raw_json_input/`
- **Description**: Runs the feature processing pipeline given a raw JSON input. The input should follow the structure of the `ECGBatch` Pydantic model.
- **Function**: `process_features_by_raw_json_input`
- **Input**: JSON data with the structure of the `ECGBatch` model.
- **Output**: JSON response with the supervisor information, record date, configurations, and extracted features.

Input Example:
![raw_json.png](app/figures/raw_json.png)
Output Example:
![raw_json output.png](app/figures/raw_json output.png)

### 2. Process Features by File Input 📂 (🔖[feature processing])

- **Endpoint**: `/file_input/`
- **Description**: Runs the feature processing pipeline given multiple inputs. It accepts HDF5 files containing ECG data and other necessary information.
- **Function**: `process_features_by_file_input`
- **Input**: Multipart form data with the following fields: `output_format`, `supervisor`, `configs`, `subject_ids`, `ecg_files`, and `labels`.
- **Output**: The output depends on the specified `output_format`. It can be a JSON response, a CSV file, or an Excel spreadsheet.

Input Example Gif:
![VideoToGif-230716-180041.gif](app/figures/VideoToGif-230716-180041.gif)
Output Example (Excel, since JSON is the same as in the previous endpoint):
![excel_output_1.png](app/figures/excel_output_1.png)

---

The pipeline workflow in combination with these endpoints allows for efficient processing of ECG data, feature extraction, and output generation.

## Requirements

See in file [requirements.txt](requirements.txt).

## Setup

### Setting it up as localhost (using pip)

Make sure you have **[python 3.10](https://www.python.org/downloads/release/python-3100/) installed** (pip comes with it in 
the installation): `python --version` and `pip --version`!

1. Clone the repo:
   ```
   $ git clone link-to-repo
   ```
2. Navigate to the repo directory:
   ```
   $ cd pipeline_hrv-02
   ```
3. Create a virtual environment (using python 3.10):
   ```
   $ python3.10 -m venv venv
   ```
4. Activate the virtual environment:
   ```
   $ source venv/bin/activate
   ```
5. Install requirements from requirements.txt: 
   ```
   $ pip install -r requirements.txt
   ```
6. Navigate to the app directory:
   ```
   $ cd app
   ```
7. Run the app:
   ```
   $ uvicorn main:app --reload
   ```

You can now access the app at http://127.0.0.1:8000 in your browser of choice. For browsing the API, you can use the 
automatic interactive API documentation provided by [Swagger UI](https://github.com/swagger-api/swagger-ui) at 
http://127.0.0.1:8000/docs or the automatic docs provided by [ReDoc](https://github.com/Redocly/redoc) at 
http://127.0.0.1:8000/redoc.

### Setting it up as docker

Make sure you have **[docker](https://docs.docker.com/get-docker/) installed**: `docker --version`!

#### 1. Building an image from the docker file
```
sudo docker build . diarization
```

#### 2. Start the docker container from the image
##### in foreground
```
sudo docker run -p8000:8000 diarization:latest
```

##### in forebackground
```
sudo docker run -d -p8000:8000 diarization:latest
```

## Repo Structure

Overall structure of the repo:

```
pipeline_hrv-02
   ┣ 📂app                     <-- The main app package.
   ┃  ┣ 📂data                 <-- Data used as input during development with Jupyter notebooks and for pydantic models as examples. 
   ┃  ┣ 📂figures              <-- Figures used in the README.md file.
   ┃  ┣ 📂models               <-- Machine learning models used during development.
   ┃  ┣ 📂notebooks            <-- Jupyter Notebooks used in development. Self-explanatory.
   ┃  ┣ 📂src                  <-- The customized project packages containing all utility functions and source codes.
   ┃  ┗ 📜main.py              <-- The final FastAPI App. 
   ┣ 📜.dockerignore           <-- Dockerignore file. 
   ┣ 📜.gitignore              <-- Gitignore file. 
   ┣ 📜Dockerfile              <-- Dockerfile for building the docker image.
   ┣ 📜README.md               <-- The top-level README for developers using this project. 
   ┣ 📜requirements.in         <-- The requirements file for reproducing the environment
   ┗ 📜requirenments.txt       <-- The requirenments file for reproducing the environment, e.g. generated with 
                                   'pip freeze > requirenments.txt'.
```

Moreover, the src contains the main source code of the app. This package includes the following subpackages:

- `configs.py` contains the configuration of the app. Mainly the static parameters of the app are defined here.
- `conversion.py` contains the source code for the conversion of the data, e.g. from hdf5 file to pandas dataframe.
- `ecg_feature_extraction.py` contains the source code for the feature extraction.
- `ecg_preprocessing.py` contains the source code for the preprocessing of the raw ecg data.
- `ecg_processing.py` contains the source code for the complete processing of the data. From raw ecg data up to the features.
- `pydantic_models.py` contains the pydantic models for the input and output data of the fastAPI pipline.
- `utils.py` contains the utility functions used in the app.

## Documentation
Link to the documentation of the libraries used in this project:
* [NeuroKit2](https://neurokit2.readthedocs.io/en/latest/)
* [FastAPI](https://fastapi.tiangolo.com/)
* [Pydantic](https://pydantic-docs.helpmanual.io/)
* [Docker](https://docs.docker.com/)
