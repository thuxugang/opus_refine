# OPUS-Refine

Protein backbone torsion angles (Phi and Psi) are crucial for protein local conformation description. In this paper, we propose a general post-processing method for all prediction methods, namely OPUS-Refine, which may contribute to the field in a different way. OPUS-Refine is a sampling-based method, therefore, the results of other prediction methods can be used as its constraints. After OPUS-Refine refinement, for instance, the accuracy of Phi/Psi predicted by SPIDER3 and SPOT-1D are both increased. In addition, to facilitate the sampling efficiency, we construct a neighbor-dependent statistical torsion angles sampling database, namely OPUS-TA, which may be useful for other sampling-based methods. Furthermore, we also introduce the contact map predicted by RaptorX to OPUS-Refine as a global structural constraint. After refinement, comparing to the predicted structures obtained from RaptorX online server, the accuracy of both global structural configurations (measured by TM-score and RMSD) and local structural configurations (measured by Phi/Psi) results are improved. OPUS-Refine is a highly efficient framework, it takes only about 4s to refine the torsion angles and 30s to refine the global structural of a protein with 100 residues in length on a typical desktop personal computer. Therefore, the sampling-based feature and the efficiency of OPUS-Refine offer greater potentiality for it to takes advantage of any other method to achieve a better performance.

## OPUS-TA

We divide the structures in the entire PDB by all possible overlapping segments (3, 5, 7-residues in length), then we gather the segments which have the same sequence and use the values of middle residue’s torsion angles as their features. Gaussian mixture model (GMM) is used to model the distribution. In each modeling, we assume that the values of Phi and Psi are correlated, and the number of Gaussian components in each GMM is dynamic. We set the number of components from 1 to 10 and model the distribution in order. If the standard deviation of Phi and Psi in all Gaussian components are less than 10, we use the value of current number of components for modeling. Otherwise, we try the next value in the sequence. Finally, we save the means and the covariance of GMM for this sequence into corresponding TA lookup table. We only consider the segments which appear at least 5 times in entire PDB.

For protein backbone torsion angles sampling, we first divide the target structure by all possible overlapping segments (3, 5, 7-residues in length). Then, for every segment findable in corresponding TA lookup table, we extract its GMM parameters. Noted that, if a residue has more than one set of GMM parameters (means and covariance) extracted from the models modeling by different length of segments, we only keep the GMM parameters derived from the longest one. Thus, each residue would have only one set of GMM parameters.

### Usage

1. Download OPUS-TA databases.

   The three OPUS-TA databases (3, 5, 7-residues in length) we used are saved in the MongoDB, the exported collections are hosted on [Baidu Drive](https://pan.baidu.com/s/1hjHMGkVfA3kM9QPQi126-w) with password `2kw9`.

2. Import OPUS-TA into your MongoDB.
   ```
   ./mongoimport -d ta_db -c ta_3 --file /your_path/ta_3.dat --type json
   ./mongoimport -d ta_db -c ta_5 --file /your_path/ta_5.dat --type json
   ./mongoimport -d ta_db -c ta_7 --file /your_path/ta_7.dat --type json
   
   db.ta_3.ensureIndex({"key":1},{"unique":true})
   db.ta_5.ensureIndex({"key":1},{"unique":true})
   db.ta_7.ensureIndex({"key":1},{"unique":true})
   ```  

3.Use OPUS-TA as your sampling database.

   i. Data Format
   ```
   {"key": "GWGVD",
   "value": "67.99999999999993_27.999999999999968#1.0000009999999988_0.9999999999999989_0.9999999999999989_1.0000009999999988#0.222222222222;-59.571428571428555_-39.14285714285713#13.6734703877551_-16.510204081632647_-16.510204081632647_24.122449979591828#0.777777777778"}
   ```
   ii. Parse
   ```
   67.99999999999993_27.999999999999968#1.0000009999999988_0.9999999999999989_0.9999999999999989_1.0000009999999988#0.222222222222;
   
   67.99999999999993_27.999999999999968 -> GMM parameters phi_psi mean
   [67.99999999999993, 27.999999999999968]
   1.0000009999999988_0.9999999999999989_0.9999999999999989_1.0000009999999988 -> GMM parameters cov
   [[1.0000009999999988, 0.9999999999999989],
    [0.9999999999999989, 1.0000009999999988]]
   0.222222222222 -> occurence probability
   
   -59.571428571428555_-39.14285714285713#13.6734703877551_-16.510204081632647_-16.510204081632647_24.122449979591828#0.777777777778
   ...
   ```
   iii. Python code
   ```
   phi_sampled, psi_sampled = np.random.multivariate_normal(mean, cov)
   ```
   
## Test Sets

We used 58 proteins in Rosetta decoy set (Rosetta) and 55 proteins in I-Tasser decoy set (I-Tasser) as the modeling test sets. Therefore, the performance of OPUS-Refine can be associated with the OPUS-CSF decoy recognition ability. Noted that ‘1ogwA_’ in I-Tasser was removed because it contains uncommon residues in its main chain.

## Performance

### Using OPUS-Refine for Torsion Angles Refinement

After we introduce the results of SPOT-1D as a torsion-angle-constraining term in OPUS-Refine, the accuracy of final outputs constrained by it is better than the accuracy of its original web-server predicted results. Therefore, OPUS-Refine can be used as a post-processing refinement for other torsion angles prediction methods.

For Phi (MAE) and Psi (MAE), the smaller the better.

|Rosetta|	SPOT-1D|	OPUS-Refine + SPOT-1D	|
|:----:|:----:|:----:|
|Phi (MAE)|11.04	|8.87|
|Psi (MAE)|	12.80|9.98|

|I-Tasser|	SPOT-1D|	OPUS-Refine + SPOT-1D	|
|:----:|:----:|:----:|
|Phi (MAE)|15.20	|13.85|
|Psi (MAE)|17.48|15.44|

### Using OPUS-Refine for Structural Refinement

We compare the OPUS-Refine refinement results using both SPOT-1D and RaptorX as constraints with the predicted structures obtained from RaptorX online server. After refinement, the accuracy of both global structural configurations (measured by TM-score and RMSD) and local structural configurations (measured by Phi and Psi) are both improved comparing to the original predicted structures obtained from RaptorX online server on both test sets, showing its potentiality to be a useful structural refinement scheme. 

For TM-score, the larger the better, for RMSD, the smaller the better.

|Rosetta|RaptorX|OPUS-Refine + SPOT-1D + RaptorX|
|:----:|:----:|:----:|
|Phi (MAE)|20.05|11.81|
|Psi (MAE)|28.24|14.62|
|TM-score|0.64|0.71|
|RMSD|2.36|1.76|

|I-Tasser|RaptorX|OPUS-Refine + SPOT-1D + RaptorX|
|:----:|:----:|:----:|
|Phi (MAE)|23.39|18.02|
|Psi (MAE)|31.74|23.21|
|TM-score|0.69|0.72|
|RMSD|2.24|2.00|

### Using OPUS-Refine for Decoy Recognition

The value of total energy function of OPUS-Refine can be used for scoring the quality of a protein structure. We list the results of different methods on 4 decoy sets. 

The numbers of targets, with their native structures successfully recognized by each method, are listed in the table. The numbers in parentheses are the average Z-scores of the native structures. The larger the absolute value of Z-score, the better of our results.

||TOTAL|OPUS-CSF|OPUS-SSF|OPUS-Refine (Score)|
|:----:|:----:|:----:|:----:|:----:|
|Rosetta (3DR)|58|	51 (−3.83)	|53 (-3.98)	|54 (-3.47)|
|I-Tasser (3DR)|56|	36 (−3.47)	|38 (-3.81)	|40 (-3.12)|
|Rosetta|	58|	47 (−5.43)|	52 (-5.81)	|55 (-5.74)|
|I-Tasser|	56|	47 (−7.70)|	50 (-9.11)	|53 (-7.41)|
## Dependency

```
Docker 18.09.7
```

## Usage

1. Download OPUS-Rota2 docker image.

   The docker image we used is hosted on [Baidu Drive](https://pan.baidu.com/s/1Jey4nMyt55zwIq4hY5CQxw) with password `rxwq`.

2. Docker image preparation.
   ```
   cat opus_refine_docker* | tar xvz
   docker load -i opus_refine.tar
   docker run -it --name refine opus_refine:1.0
   docker start refine 
   docker attach refine 
   ```
   
3. Run MongoDB. 
   ```
   cd /home/mongodb/bin
   ./mongod -dbpath /home/mongodb/data/db -logpath /home/mongodb/log/mongodb.log -logappend -fork -port 27017
   ```    
4. Run OPUS-Refine.

   ```
   cd /home/opus_refine
   ./opus_refine
   ```    
   The configurations we used for Torsion Angles Refinement and Structural Refinement are placed in `configs`, you can simpliy switch and rename them with `opus_refine.ini`. We put the constrained contact map files and their corresponding list into `constains_files/contact_maps`, constrained torsional angles files and their corresponding list into `constains_files/torsion_angles` and initial structures files and their corresponding list into `constains_files/init_structures`. Our results can be found in `fold_output`.
   
5. Run OPUS-Refine Score for decoy recognition.

   ```
   cd /home/opus_refine_score
   ./opus_refine_score
   ```    
   First, you need to put each constrained file into `constains_files`, otherwise, you get OPUS-CSF score. Then, format your inputs as our examples shown in `opus_refines_list.txt`. 
   
   ```
   > 1abv_.pdb
   /home/opus_refine_score/test/1abv_.pdb
   ```
      
   The first line is the filename which is used to locate the constrained files through each file list in `constains_files`. The second line is the path.

## Reference 
```bibtex
@article{xu2020opus,
  title={OPUS-Refine: A Fast Sampling-based Framework for Refining Protein Backbone Torsion Angles and Global Conformation},
  author={Xu, Gang and Wang, Qinghua and Ma, Jianpeng},
  journal={Journal of Chemical Theory and Computation},
  year={2020},
  publisher={ACS Publications}
}
```
