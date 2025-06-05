# Beyond Classification: Inferring Function Names in Stripped Binaries via Domain Adapted LLMs
This repository contains the code implementation of paper [Beyond Classification: Inferring Function Names in Stripped Binaries via Domain Adapted LLMs](https://www.ndss-symposium.org/wp-content/uploads/2025-797-paper.pdf).

We implemented SymGen using [Ghidra](https://ghidra-sre.org/) (for decompilation), [ANTLR 4](https://www.antlr.org/) (for processing source code and decompiled code). 
The model's fine-tuning and prediction are based on [alpaca-lora](https://github.com/tloen/alpaca-lora/tree/main).
For more details, please refer to our paper.


## Repository Contents


## Environment Setup
1. Create new conda enviroment
    ```bash
    conda create -n symgen python=3.9
    ```

2. Activate the conda environment
    ```bash
    conda activate symgen
    ```

3. Install packages
    ```bash
    pip install -r requirements.txt
    ```

4. Install Ghidra

    The Ghidra installation guide can be found [here](https://htmlpreview.github.io/?https://github.com/NationalSecurityAgency/ghidra/blob/stable/GhidraDocs/InstallationGuide.html).


## Dataset
The original source code, binaries, and decompiled code used by SymGen can be downloaded [here](https://zenodo.org/records/15530083).

The uploaded package follows a three-level directory structure, organized as shown below:
```
.
├── architecture
    └── opt_level
        └── project
```
- `architecture`: Contains four architectures: x86_64, x86_32, arm, and mips.
- `opt_level`: Contains different optimization levels: O0, O1, O2, and O3.
- `project`: Contains 33 projects. The list can be found in our paper.

A processed sample dataset is provided in the folder [`dataset/`](dataset/) which can be used for fine-tuning and prediction directly (See [Fine-tune and Predict](#fine-tune-and-predict)).


## Running Steps
All python scripts are in the folder [`scripts/`](scripts/).
```bash
cd scripts/
```
To save the results at each step, we provide configurable arguments (e.g., --output_dir) that allow you to specify the desired output directory.
For convenience, you can also update the `default` values directly in the scripts.

### Decompile Binaries
The first step is to obtain the decompiled code for the functions.
The related scripts are in the folder [`scripts/decompilation`](scripts/decompilation).

1. Start by specifying the `output_dir` in [`decomp_for_unstripped.py`](scripts/decompilation/decomp_for_unstripped.py) and [`decomp_for_stripped.py`](scripts/decompilation/decomp_for_stripped.py) to define where the results will be saved. 
2. Use [`parallel_decomp.py`](scripts/decompilation/parallel_decomp.py) to decompile your binaries. 
    - Use the `-u` option for unstripped binaries 
    - Use the `-s` option for stripped binaries.

    ```bash
    python decompilation/parallel_decomp.py [-u | -s] \
        --binary_path YOUR_BINARIES_PATH \
        --ghidra_path YOUR_GHIDRA_PATH \
        --project_path YOUR_GHIDRA_PROJECT_PATH
    ```
    Arguments:
    - `-u`, `--unstripped`: Indicates that the binary is unstripped (contains debug symbols).
    - `-s`, `--stripped`: Indicates that the binary is stripped (lacks debug symbols).
    - `-b`, `--binary_path`: Specify the path to the binary file or folder containing binaries.
    - `-g`, `--ghidra_path`: Provide the path to the Ghidra *analyzeHeadless*.
    - `-p`, `--project_path`: Specify the directory path to Ghidra projects.


### Process Decompiled Function Code
After get the decompiled code, we need further process the data. 
The related scripts are in the folder [`scripts/data_processing`](scripts/data_processing).

For generating a dataset (`-d`), we need combine the stripped functions with unstripped functions using DWARF.
For prediction of stripped binary (`-p`), it only needs to output the test set file containing all masked functions. 

#### Combine the stripped funcions with unstripped funcions
For generating a dataset:
```bash
python data_processing/process_decompiled_data.py -d \
    --unstripped_path YOUR_UNSTRIPPED_DECOMP_DATA_PATH \
    --stripped_path YOUR_STRIPPED_DECOMP_DATA_PATH \
    --output_dir YOUR_OUTPUT_DIR
```

For prediction of stripped binary:
```bash
python data_processing/process_decompiled_data.py -p \
    --stripped_path YOUR_STRIPPED_DECOMP_DATA_PATH \
    --output_dir YOUR_OUTPUT_DIR
```

Arguments:
- `-d`, `--dataset`: Indicates the purpose of generating a new dataset for training and testing purposes.
- `-p`, `--prediction`: Indicates the purpose of prediction of a new stripped binary.
- `-u`, `--unstripped_path`: Path to JSON files containing decompiled unstripped binaries.
- `-s`, `--stripped_path`: Path to JSON files containing decompiled stripped binaries.
- `-o`, `--output_dir`: Directory to save the output files.


#### Divide Dataset
*For prediction of stripped binary, this substep is not required.*

Use [`divide_dataset.py`](scripts/data_processing/divide_dataset.py) to split the processed data into training, test, and validation sets. Duplicate entries are removed from the dataset based on the function name and function body. For more details, please refer to our paper.
```bash
python data_processing/divide_dataset.py \
    --input_dir YOUR_PROCESSED_DECOMP_DIR \
    --output_dir YOUR_OUTPUT_DIR
```
Arguments:
- `-i`, `--input_dir`: Directory containing the combined decompiled code.
- `-o`, `--output_dir`: Directory to save the divided dataset.


### Generate Function Summary
*For prediction of stripped binary, this step is not required.* 

We provide a partial summary generated by LLM, which can be found in the folder [dataset/](dataset/).

Extract source function code from your source project (`--source_dir`) and generate function summaries for fine-tuning.
The related scripts are in the folder [`scripts/summary_generation`](scripts/summary_generation).

#### Extract Function from Source Code Project
To extract functions from the source project code, use the following command:
```bash
python summary_generation/extract_functions.py \
    --source_dir YOUR_SOURCE_PROJECT_DIR \
    --test_file YOUR_TESTSET_FILE \
    --output_dir YOUR_OUTPUT_DIR
```
Arguments:
- `-s`, `--source_dir`: Directory containing the source projects.
- `-t`, `--test_file`: Path to the test set file for removing duplicates in the collected source function code. For example: /dataset/test_set.json.'
- `-o`, `--output_dir`: Directory to save the output JSON file.


#### Generate Function Summary
To generate summaries for the extracted source functions, use the following command:
```bash
CUDA_VISIBLE_DEVICES=0 python summary_generation/generate_summary.py \
    --base_model BASE_MODEL \
    --input_file YOUR_SOURCE_FUNC_FILE \
    --output_dir YOUR_OUTPUT_DIR
```
Arguments:
- `--base_model`: Specify a base model, e.g. --base_model='codellama/CodeLlama-34b-Instruct-hf'.
- `--input_file`: Path to the extracted source function file.
- `--output_dir`: Directory to save the output JSON file.


#### Merge Training Set with Function Summary Data
To merge the generated summaries and original training set, use the following command:
```bash
python summary_generation/merge_data.py \
    --training_data YOUR_TRAINING_SET_FILE \
    --summary_data YOUR_SUMMARY_DATA_FILE \
    --output_dir YOUR_OUTPUT_DIR
```
Arguments:
- `-t`, `--training_data`: Path to the JSON file containing the decompiled training set.
- `-s`, `--summary_data`: Path to the JSON file containing function summaries.
- `-o`, `--output_dir`: Directory to save the merged training set.


### Fine-tune and Predict
This step is implemented based on [alpaca-lora](https://github.com/tloen/alpaca-lora/tree/main).

We use the Code Llama prompt format (see [templates/codellama.json](scripts/model_training/templates/codellama.json)) and adapt the prediction process in [`predict.py`](scripts/model_training/predict.py) for our task.
The related scripts are in the folder [`scripts/model_training`](scripts/model_training).

#### Fine-tune
To fine-tune the model using your decompiled training set, use the following command:
```bash
CUDA_VISIBLE_DEVICES=0 python model_training/finetune.py \
    --base_model BASE_MODEL \
    --data_path YOUR_TRAINING_SET_FILE \
    --output_dir YOUR_OUTPUT_DIR
```
Arguments:
- `--base_model`: Specify a base model, e.g. --base_model='codellama/CodeLlama-34b-Instruct-hf'.
- `--data_path`: Path to the JSON file containing the training set.
- `--output_dir`: Directory to save the lora weight.

You can also use the LoRA weights in the folder [lora_weights/](lora_weights)


#### Predict
To generate predictions using a fine-tuned model, execute the following command:
```bash
CUDA_VISIBLE_DEVICES=0 python model_training/predict.py \
    --base_model BASE_MODEL \
    --lora_weights SAVED_LORA_WEIGHT_DIR \
    --input_path YOUR_TEST_SET_FILE \
    --output_dir YOUR_OUTPUT_DIR
```
Arguments:
- `--base_model`: Specify a base model, e.g. --base_model='codellama/CodeLlama-34b-Instruct-hf'.
- `--lora_weights`: Directory of the saved lora weight.
- `--input_path`: Path to the JSON file containing the test set.
- `--output_dir`: Directory to save the prediction results.


### Evaluate
The related scripts are in the folder [`scripts/evaluation`](scripts/evaluation).

1. Extract the function name from the results and divide them into tokens:
    ```bash
    python evaluation/divide_function_name.py \
        --input_file YOUR_PRED_RESULTS_FILE \
        --output_dir YOUR_OUTPUT_DIR
    ```
    Arguments:
    - `-i`, `--input_file`: Path to the input file containing predicted function names and ground truth.
    - `-o`, `--output_dir`: Directory to save the processed evaluation results.


2. To calculate the precision, recall and F1 score, use the following command:
    ```bash
    python evaluation/calculate_score.py \
        --input_file YOUR_PROCESSED_RESULTS_FILE
    ```
    Arguments:
    - `-i`, `--input_file`: Path to the evaluation input file.


## License
This project is licensed under the Apache License 2.0 (See [LICENSE](LICENSE)).

This project includes other components distributed under the Apache License 2.0:

- alpaca-lora (See [scripts/model_training/LICENSE](scripts/model_training/LICENSE))
