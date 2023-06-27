# Data quality checks

Quality checks play a crucial role in ensuring the reliability and validity of data in various fields, including MRI imaging. These checks are designed to assess the quality of acquired data and identify potential issues or errors that may affect the accuracy and interpretability of the results. By conducting quality checks, researchers and clinicians can evaluate the overall data quality, detect artifacts, noise, or other sources of error, and make informed decisions regarding the suitability of the data for further analysis or clinical diagnosis. Quality checks also help in identifying and addressing technical or procedural issues that may impact data collection, ensuring that the data meets the necessary standards for research or clinical applications. Ultimately, quality checks contribute to enhancing the robustness and credibility of findings derived from the data, enabling researchers and clinicians to draw accurate conclusions and make informed decisions based on high-quality data.

---

## Automated assessment of MRI data quality
In this guide, we will explore the use of two tools, [MRIqc](https://github.com/nipreps/mriqc) and [eddyqc](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddyqc/UsersGuide), for automated assessment of MRI data quality.

| MRI modality  | Tool           
| ------------- |:-------------:|
| T1w           | mriqc         | 
| T2w           | mriqc         |   
| fMRI          | mriqc         |  
| DWI           | eddyqc        |  

---

### MRIqc

MRIqc is a tool that requires data to be in BIDS compliant format. You can find a useful guide on how to construct a BIDS dataset [here](https://andysbrainbook.readthedocs.io/en/latest/OpenScience/OS/BIDS_Overview.html). 

This tutorial is designed to run on a high-performance computing system (HPC). If you need basic information about HPC tailored to the neuroimaging community, you can refer to the documentation [here](https://neuroimaging-core-docs.readthedocs.io/en/latest/pages/hpc.html#). We will use a containerized version of MRIqc with Singularity >2.5.

#### Building MRIqc singulaity container
Once you are logged in to the HPC, you can either start an interactive session or run an sbatch script. Here, we will go with the second option. Remember to replace `version` with the desired version.

```bash
#!/bin/sh

#SBATCH -p high
#SBATCH --mail-user=your_mail_address
#SBATCH -N 1
#SBATCH --output=mriqcbuild_%j.out
#SBATCH --error=mriqcbuild_%j.err
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=8G

singularity build $HOME/mriqc-<version>.simg docker://poldracklab/mriqc:<version>
```
#### Running MRIqc
Below is an sbatch script to run MRIQC. The necessary user inputs are highlighted in a separate section. You can run MRIqc on a specific MRI modality only (e.g., T1w) or on all the compatible data modalities in your BIDS directory.

```bash
#!/bin/sh
#SBATCH -p high
#SBATCH --mail-user=your_mail_address
#SBATCH -N 1
#SBATCH --output=mriqc_%j.out
#SBATCH --error=mriqc_%j.err
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=8G

# -----------------------------------------------------------------------------------------------------------------------------------------------------
# User inputs:

singularity_image="/path/to/mriqc.simg"  # Path to the MRIqc Singularity container image
bids_dir="/path/to/bids/dataset"  # Root directory of your BIDS dataset

subj=[sub01 sub02 sub03]  # List of subjects to process

multimodal=no  # Flag indicating whether to perform QC on all MRI modalities (yes) or a specific modality of choice (no)
modality=T1w  # The MRI modality to perform QC on when multimodal=no (T1w, T2w, or bold)
# -----------------------------------------------------------------------------------------------------------------------------------------------------

# Make mriqc directory
if [ ! -d $bids_dir/derivatives/mriqc ]; then
    mkdir $bids_dir/derivatives/mriqc  # Create a directory to store the MRIQC results if it doesn't exist
fi

# Run MRIQC
echo "Running MRIQC"

if [ $multimodal == yes ]; then
    unset PYTHONPATH; singularity run $singularity_image \
    $bids_dir $bids_dir/derivatives/mriqc \
    participant \
    --participant-label $subj \
    done
else
    for subj_i in subj; do
        unset PYTHONPATH; singularity run $singularity_image \
        $bids_dir $bids_dir/derivatives/mriqc \
        participant \
        --participant-label $subj \
        -m $modality /
    done
fi

echo "MRIQC analysis complete!"
```
<table><tr><td>Group level can be run once having output from the single level.</td></tr></table>

---

### Eddyqc
`eddy` is a part of the FSL libraries used to remove eddy currents from DWI data. With a specific code tag, it can also provide QC. 

In the sample code below, we will be running eddyqc as part of the [MRtrix](https://mrtrix.readthedocs.io/en/latest/) workflow. 

The default eddy QC, called upon by `-eddyqc_text eddyqc`, are provided as a PDF file and as part of the JSON file. To perform additional QC metrics such as SNR or CNR, use `--cnr_maps` tag.

```bash
dwifslpreproc dwi.nii.gz dwi_eddy.nii.gz \
-fslgrad dwi.bvecs dwi.bvals -rpe_none -pe_dir AP -eddy_options " --cnr_maps --repol --slm=linear --data_is_shelled" -eddyqc_text eddyqc
```

Need help interpreting the QC metrics? [Here](https://rpubs.com/navona/SPINS_DWI_QCautomated) you can find some suggested thresholds to start working from.