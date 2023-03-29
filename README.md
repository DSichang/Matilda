# Matilda: Multi-task learning from single-cell multimodal omics

Matilda is a multi-task framework for learning from single-cell multimodal omics data. Matilda leverages the information from the multi-modality of such data and trains a neural network model to simultaneously learn multiple tasks including data simulation, dimension reduction, visualization, classification, and feature selection.

<img width=100% src="https://github.com/liuchunlei0430/Matilda/blob/main/img/main.jpg"/>

## Installation
Matilda is developed using PyTorch 1.9.1 and requires >=1 GPU to run. We recommend using conda enviroment to install and run Matilda. We assume conda is installed. You can use the provided environment or install the environment by yourself accoring to your hardware settings. Note the following installation code snippets were tested on a Ubuntu system (v20.04) with NVIDIA GeForce 3090 GPU. The installation process needs about 15 minutes.

### Installation using provided environment

Step 1:
Otain Matilda by clonning the github repository:
```
git clone https://github.com/PYangLab/Matilda.git
```

Step 2: Create and activate the conda environment for matilda using our provided file
```
cd Matilda
conda env create -f environment_matilda.yaml
conda activate environment_matilda
```

Step 3:
The following python packages are required for running Matilda: h5py, numpy, pandas, captum. They can be installed in the conda environment as below:
```
pip install h5py
pip install numpy
pip install pandas
pip install captum
pip install tqdm
pip install scipy
pip install scanpy
```



### Installation by youself

Step 1:
Create and activate the conda environment for matilda
```
conda create -n environment_matilda python=3.7
conda activate environment_matilda
```

Step 2:
Check the environment including GPU settings and the highest CUDA version allowed by the GPU.
```
nvidia-smi
```

Step 3:
Install pytorch and cuda version based on your GPU settings.
```
# Example code for installing CUDA 11.3
conda install pytorch==1.9.1 torchvision==0.10.1 torchaudio==0.9.1 cudatoolkit=11.3 -c pytorch -c conda-forge
```

Step 4:
The following python packages are required for running Matilda: h5py, numpy, pandas, captum. They can be installed in the conda environment as below:
```
pip install h5py
pip install numpy
pip install pandas
pip install captum
pip install tqdm
pip install scipy
pip install scanpy
```

Step 5:
Otain Matilda by clonning the github repository:
```
git clone https://github.com/PYangLab/Matilda.git
```

## Preparing intput for Matilda
Matilda’s main function takes expression data (e.g., RNA, ADT, ATAC) in `.h5` format and cell type labels in `.csv` format. Matilda expects raw count data for RNA and ADT modalities. For ATAC modality, Matilda expects the 'gene activity score' generated by Seurat from raw count data.

An example for creating .h5 file from expression matrix in the R environment is as below:
```
write_h5 <- function(exprs_list, h5file_list) {  
  for (i in seq_along(exprs_list)) {
    h5createFile(h5file_list[i])
    h5createGroup(h5file_list[i], "matrix")
    writeHDF5Array(t((exprs_list[[i]])), h5file_list[i], name = "matrix/data")
    h5write(rownames(exprs_list[[i]]), h5file_list[i], name = "matrix/features")
    h5write(colnames(exprs_list[[i]]), h5file_list[i], name = "matrix/barcodes")
  }  
}
write_h5(exprs_list = list(rna = train_rna, h5file_list = "/Matilda/data/TEA-seq/train_rna.h5")
```

An example for creating gene activity score from ATAC modality in the R environment using human gene annotation is as below:
```
gene.activities <- CreateGeneActivityMatrix2(peak.matrix=teaseq.peak,
                                             annotation.file = “Homo_sapiens.GRCh38.90.chr.gtf.gz”,
                                             seq.levels = c(1:22, “X”, “Y”),
                                             seq_replace = c(“:”))
```

### Example dataset


As an example, the processed TEA-seq dataset by Swanson et al. (GSE158013) is provided for the example run, which is saved in `./Matilda/data/TEAseq`.
Users can prepare the example dataset as input for Matilda or use their own datasets.
Training and testing on demo dataset will cost no more than 1 minute with GPU.

## Running Matilda with the example dataset
### Training the Matilda model (see Arguments section for more details). If we use multimodality data, we use `main_matilda_train.py`; if we use scRNA-seq data, we use `main_matilda_rna_train.py`.
```
cd Matilda
cd main
# training the matilda model
python main_matilda_train.py --rna [trainRNA] --adt [trainADT] --atac [trainATAC] --cty [traincty] #[training dataset]
# Example run
python main_matilda_train.py --rna ../data/TEAseq/train_rna.h5 --adt ../data/TEAseq/train_adt.h5 --atac ../data/TEAseq/train_atac.h5 --cty ../data/TEAseq/train_cty.csv
# python main_matilda_rna_train.py --rna ../data/TEAseq/train_rna.h5 --cty ../data/TEAseq/train_cty.csv # for scRNA-seq
```
### Argument
Training dataset information
+ `--rna`: path to training data RNA modality.
+ `--adt`: path to training data ADT modality (can be null if ATAC is provided).
+ `--atac`: path to training data ATAC modality (can be null if ADT is provided). Note ATAC data should be summarised to the gene level as "gene activity score".
+ `--cty`: path to the labels of training data.

Training and model config
+ `--batch_size`: Batch size (set as 64 by default)
+ `--epochs`: Number of epochs.
+ `--lr`: Learning rate.
+ `--z_dim`: Dimension of latent space.
+ `--hidden_rna`: Dimension of RNA branch.
+ `--hidden_adt`: Dimension of ADT branch.
+ `--hidden_atac`: Dimension of ATAC branch.

Other config
+ `--seed`: The random seed for training.
+ `--augmentation`: Whether to augment simulated data.

Note: after training, the model will be saved in `./Matilda/trained_model/`.



### Perform multiple tasks using trained Matilda model.
After training the model, we can use `main_matilda_task.py` to do multiple tasks with different augments. For scRNA-seq data, we can use `main_matilda_rna_task.py`.

### Argument for performing tasks
+ `--classification`: whether to do cell type classification.
+ `--fs`: whether to do cell type feature selection.
+ `--dim_reduce`: whether to do dimension reduction.
+ `--simulation`: whether to do simulation. 
+ `--simulation_ct`: an integer index for which cell type to simulate. Only be activated when `simulation = True`.
+ `--simulation_num`: the number of cells to simulate for the specified cell type. Only be activated when `simulation = True`.


**1) Multi-task on the training data**

```
# using the trained model for data simulation
python main_matilda_task.py  --rna [trainRNA] --adt [trainADT] --atac [trainATAC] --cty [traincty] --simulation True --simulation_ct 1 --simulation_num 200
# Example run
python main_matilda_task.py --rna ../data/TEAseq/train_rna.h5 --adt ../data/TEAseq/train_adt.h5 --atac ../data/TEAseq/train_atac.h5 --cty ../data/TEAseq/train_cty.csv --simulation True --simulation_ct 1 --simulation_num 200
# python main_matilda_rna_task.py --rna ../data/TEAseq/train_rna.h5 --cty ../data/TEAseq/train_cty.csv --simulation True --simulation_ct 1 --simulation_num 200  # for scRNA-seq
```
Output: The output will be saved in `./Matilda/output/simulation_result/TEAseq/reference/`. To generate UMAP plots for the simulated data using R, run `./Matilda/qc/visualize_simulated_data.Rmd`. The UMAPs are:

<img width=60% src="https://github.com/liuchunlei0430/Matilda/blob/main/img/simulation_anchor.jpg"/> 

```
# using the trained model for data dimension reduction and visualisation
python main_matilda_task.py  --rna [trainRNA] --adt [trainADT] --atac [trainATAC] --cty [traincty] --dim_reduce True
# Example run
python main_matilda_task.py --rna ../data/TEAseq/train_rna.h5 --adt ../data/TEAseq/train_adt.h5 --atac ../data/TEAseq/train_atac.h5 --cty ../data/TEAseq/train_cty.csv --dim_reduce True
# python main_matilda_rna_task.py --rna ../data/TEAseq/train_rna.h5 --cty ../data/TEAseq/train_cty.csv --dim_reduce True  # for scRNA-seq
```
Output: The output will be saved in `./Matilda/output/dim_reduce/TEAseq/reference/`. To generate UMAP plots and 4 clustering metrices, i.e., ARI, NMI, FM, Jaccard, for the latent space using R, run `./Matilda/qc/visualize_latent_space.Rmd`. The UMAPs are:

<img width=50% src="https://github.com/liuchunlei0430/Matilda/blob/main/img/visualisation.jpg"/> 

```
# using the trained model for feature selection
python main_matilda_task.py  --rna [trainRNA] --adt [trainADT] --atac [trainATAC] --cty [traincty] --fs True
# Example run
python main_matilda_task.py --rna ../data/TEAseq/train_rna.h5 --adt ../data/TEAseq/train_adt.h5 --atac ../data/TEAseq/train_atac.h5 --cty ../data/TEAseq/train_cty.csv --fs True
# python main_matilda_rna_task.py --rna ../data/TEAseq/train_rna.h5  --cty ../data/TEAseq/train_cty.csv --fs True  # for scRNA-seq
```
Output: The output, i.e. feature importance scores, will be saved in `./Matilda/output/marker/TEAseq/reference/`. 


**2) Multi-task on the query data**
```
# using the trained model for classifying query data
python main_matilda_task.py  --rna [queryRNA] --adt [queryADT] --atac [queryATAC] --cty [querycty] --classification True
# Example run
python main_matilda_task.py --rna ../data/TEAseq/test_rna.h5 --adt ../data/TEAseq/test_adt.h5 --atac ../data/TEAseq/test_atac.h5 --cty ../data/TEAseq/test_cty.csv --classification True --query True
# python main_matilda_rna_task.py --rna ../data/TEAseq/test_rna.h5 --cty ../data/TEAseq/test_cty.csv --classification True --query True # for scRNA-seq
```

Output: The output will be saved in `./Matilda/output/classification/TEAseq/query/`.

```
cell ID:  0 	 	 real cell type: T.CD4.Memory 	 	 predicted cell type: T.CD4.Naive 	 	 probability: 0.77
cell ID:  1 	 	 real cell type: B.Activated 	 	   predicted cell type: B.Activated 	 	 probability: 0.53
cell ID:  2 	 	 real cell type: B.Naive 	 	       predicted cell type: B.Naive 	 	     probability: 0.73
cell ID:  3 	 	 real cell type: T.CD4.Naive 	 	   predicted cell type: T.CD4.Naive 	 	 probability: 0.78
cell ID:  4 	 	 real cell type: T.CD4.Memory 	 	 predicted cell type: T.CD4.Memory 	 	 probability: 0.87
cell ID:  5 	 	 real cell type: Mono.CD14 	 	     predicted cell type: Mono.CD14 	 	   probability: 0.95
cell ID:  6 	 	 real cell type: B.Naive 	 	       predicted cell type: B.Naive 	 	     probability: 0.78
cell ID:  7 	 	 real cell type: Mono.CD14 	 	     predicted cell type: Mono.CD14 	 	   probability: 0.96
cell ID:  8 	 	 real cell type: T.CD8.Effector 	 predicted cell type: T.CD8.Effector 	 probability: 0.95
……
```

```
cell type ID:  0                 cell type: B.Activated          prec : tensor(72.2454, device='cuda:0') number: 180
cell type ID:  1                 cell type: B.Naive              prec : tensor(98.1400, device='cuda:0') number: 802
cell type ID:  2                 cell type: DC.Myeloid           prec : tensor(40., device='cuda:0') number: 11
cell type ID:  3                 cell type: Mono.CD14            prec : tensor(98.6156, device='cuda:0') number: 639
cell type ID:  4                 cell type: Mono.CD16            prec : tensor(74.1379, device='cuda:0') number: 37
cell type ID:  5                 cell type: NK                   prec : tensor(97.1820, device='cuda:0') number: 283
cell type ID:  6                 cell type: Platelets            prec : tensor(45.4545, device='cuda:0') number: 12
cell type ID:  7                 cell type: T.CD4.Memory         prec : tensor(73.3831, device='cuda:0') number: 1189
cell type ID:  8                 cell type: T.CD4.Naive          prec : tensor(76.2363, device='cuda:0') number: 1020
cell type ID:  9                 cell type: T.CD8.Effector       prec : tensor(83.4451, device='cuda:0') number: 576
cell type ID:  10                cell type: T.CD8.Naive          prec : tensor(84.5635, device='cuda:0') number: 299
```


```
# using the trained model for dimension reduction and visualising query data
python main_matilda_task.py --rna [queryRNA] --adt [queryADT] --atac [queryATAC] --cty [querycty] --dim_reduce True
# Example run
python main_matilda_task.py  --rna ../data/TEAseq/test_rna.h5 --adt ../data/TEAseq/test_adt.h5 --atac ../data/TEAseq/test_atac.h5 --cty ../data/TEAseq/test_cty.csv --dim_reduce True --query True
# python main_matilda_rna_task.py  --rna ../data/TEAseq/test_rna.h5  --cty ../data/TEAseq/test_cty.csv --dim_reduce True --query True # for scRNA-seq
```

Output: The output will be saved in `./Matilda/output/dim_reduce/TEAseq/query/`. To generate UMAP plots and 4 clustering metrices, i.e., ARI, NMI, FM, Jaccard, for the latent space using R, run `./Matilda/qc/visualize_latent_space.Rmd`. The UMAPs are:

<img width=50% src="https://github.com/liuchunlei0430/Matilda/blob/main/img/visualisation2.png"/>  

```
# using the trained model for feature selection
python main_matilda_task.py --rna [queryRNA] --adt [queryADT] --atac [queryATAC] --cty [querycty] --fs True
# Example run
python main_matilda_task.py  --rna ../data/TEAseq/test_rna.h5 --adt ../data/TEAseq/test_adt.h5 --atac ../data/TEAseq/test_atac.h5 --cty ../data/TEAseq/test_cty.csv  --fs True --query True
# python main_matilda_rna_task.py  --rna ../data/TEAseq/test_rna.h5  --cty ../data/TEAseq/test_cty.csv  --fs True --query True # for scRNA-seq
```

Output: The output, i.e. feature importance scores, will be saved in `./Matilda/output/markers/TEAseq/query/`. 


## Reference

To cite Matilda:

[1] Chunlei Liu, Hao Huang, Pengyi Yang. Multi-task learning from multimodal single-cell omics with Matilda. Nucleic Acids Research, doi: 10.1093/nar/gkad157, (2023).


Other references:

[2] Ramaswamy, A. et al. Immune dysregulation and autoreactivity correlate with disease severity in
SARS-CoV-2-associated multisystem inflammatory syndrome in children. Immunity 54, 1083–
1095.e7 (2021).

[3] Ma, A., McDermaid, A., Xu, J., Chang, Y. & Ma, Q. Integrative Methods and Practical Challenges
for Single-Cell Multi-omics. Trends Biotechnol. 38, 1007–1022 (2020).

[4] Swanson, E. et al. Simultaneous trimodal single-cell measurement of transcripts, epitopes, and
chromatin accessibility using TEA-seq. Elife 10, (2021).

## License

This project is covered under the Apache 2.0 License.

