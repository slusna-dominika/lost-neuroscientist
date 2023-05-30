In the following guide, we will go through the following steps:

1. [Basic preprocessing](#basic-preprocessing)
2. [Estimate the response function](#response-function-estimation)
3. [Create a GM/WM boundary for seed analysis](#create-a-gmwm-boundary-for-seed-analysis)

---

## Basic preprocessing
```bash
echo "-- Step (1) Converting data to MIF format."
	mrconvert dwi.nii.gz dwi.mif -fslgrad dwi.bvec dwi.bval

echo "-- Step (2) Denoising the data"
	dwidenoise dwi.mif dwi_den.mif -noise noise.mif

	# 	Calculate the residuals:  mrcalc dwi.mif dwi_den.mif -subtract residual.mif
	# 							  mrview residual.mif
	# 	Remove Gibb's ringing artifacts (if necessary):	mrdegibbs dwi_den.mif dwi_den_unr.mif

echo "-- Step (3) Preprocessing the data"
	#    dwipreproc has 4 different modes of operation. If "no variation in phase encoding" (-rpe_none) was acquired, then specify only the AP direction (-pe_dir AP).
	#    Adding eddy options "repol" (remove outlier slices), "slm=linear" (if diff. gradients are < 60, it models their eddy currents), "data_is_shelled" (bypass the check of data as shelled).
	dwifslpreproc dwi_den.mif  dwi_den_preproc.mif -rpe_none -pe_dir AP -eddy_options " --repol --slm=linear --data_is_shelled" 

	#	Check the output via: mrview dwi_den_preproc.mif -overlay.load dwi.mif

echo "-- Step (4) # Performs bias field correction. Needs ANTs to be installed in order to use the "ants" option (use "fsl" otherwise)"
	dwibiascorrect ants dwi_den_preproc.mif dwi_den_preproc_unbiased.mif -bias bias.mif

echo "-- Step (5) Generate a mask for future processing steps"
	dwi2mask dwi_den_preproc_unbiased.mif mask.mif
```

---

## Response function estimation
Multi-shell data:
```bash
echo "-- Step (6) # Create a basis function from the subject's DWI data."
	dwi2response dhollander dwi_den_preproc_unbiased.mif wm.txt gm.txt csf.txt -voxels voxels.mif

	# 	To view what voxels were used to construct the basis function, do: mrview dwi_den_preproc_unbiased.mif -overlay.load voxels.mif
	# 	Check the response function for issue tissue type: 
	#														shview wm.txt
	#														shview gm.txt
	#														shview csf.txt

echo "-- Step (7) Create fiber orientation densities (FODs) via multishell-multitissue constrained spherical deconvolution, using the basis functions estimated above."
	dwi2fod msmt_csd dwi_den_preproc_unbiased.mif -mask mask.mif wm.txt wmfod.mif gm.txt gmfod.mif csf.txt csffod.mif

echo "-- Step (8) Combine FODs into a single image where fiber orientation densities are overlaid onto the estimated tissues (Blue=WM; Green=GM; Red=CSF)."
	mrconvert -coord 3 0 wmfod.mif - | mrcat csffod.mif gmfod.mif - vf.mif

	# 	View the FODs: 	mrview vf.mif -odf.load_sh wmfod.mif

echo "-- Step (9) Normalize the FODs for later group analyses"
	mtnormalise wmfod.mif wmfod_norm.mif gmfod.mif gmfod_norm.mif csffod.mif csffod_norm.mif -mask mask.mif

echo "-- Step (10) Convert the anatomical image"
	mrconvert T1raw.nii.gz T1.mif
```

Single-shell data:
```bash
echo "-- Step (6) # Create a basis function from the subject's DWI data."
    dwi2response dhollander dwi_den_preproc_unbiased.mif RF_WM_DHol.txt RF_GM_DHol.txt RF_CSF_DHol.txt -fslgrad dwi.bvecs dwi.bvals -mask mask.mif

echo "-- Step (7) Create fiber orientation densities (FODs)"
	dwi2fod msmt_csd dwi_den_preproc_unbiased.mif RF_WM_DHol.txt WM_FODs.nii.gz RF_CSF_DHol.txt CSF_FODs.nii.gz -fslgrad dwi.bvecs dwi.bvals -mask mask.mif
  
echo "-- Step (8) Combine FODs"
    mrconvert -coord 3 0 WM_FODs.mif - | mrcat CSF_FODs.mif - vf.mif

echo "-- Step (9) Normalize FODs"
    mtnormalise WM_FODs.mif normWM_FODs.mif CSF_FODs.mif normCSF_FODs.mif -mask mask.mif

echo "-- Step (10) Convert the anatomical image"
	mrconvert T1raw.nii.gz T1.mif
```

## Create a GM/WM boundary for seed analysis 
```bash
echo "-- Step (11) Segment the anatomical image to different tissue types for anatomically constrained tractography (1=GM; 2=Subcortical GM; 3=WM; 4=CSF; 5=Pathological tissue)."
	5ttgen fsl T1.mif 5tt_nocoreg.mif

echo "-- Step (12) Extract the average of the b0 images from the diffusion data (which has the best contrast) for later coregistration."
	dwiextract dwi_den_preproc_unbiased.mif - -bzero | mrmath - mean mean_b0.mif -axis 3

echo "-- Step (13) Converting the images to nifti (since the following commands work in FSL)."
	mrconvert mean_b0.mif mean_b0.nii.gz
	mrconvert 5tt_nocoreg.mif 5tt_nocoreg.nii.gz

echo "-- Step (14) Extracting first volume of the segmented dataset."
	fslroi 5tt_nocoreg.nii.gz 5tt_vol0.nii.gz 0 1

echo "-- Step (15) Coregistering DWI and T1 images via FSL."
	flirt -in mean_b0.nii.gz -ref 5tt_vol0.nii.gz -interp nearestneighbour -dof 6 -omat diff2struct_fsl.mat

echo "-- Step (16) Converting the transformation matrix to MRtrix-readable format."
	transformconvert diff2struct_fsl.mat mean_b0.nii.gz 5tt_nocoreg.nii.gz flirt_import diff2struct_mrtrix.txt

echo "-- Step (17) Transfer the anatomical image to the diffusion image."
	mrtransform 5tt_nocoreg.mif -linear diff2struct_mrtrix.txt -inverse 5tt_coreg.mif

	#	Examine the quality of coregistration via: mrview dwi_den_preproc_unbiased.mif -overlay.load 5tt_nocoreg.mif -overlay.colourmap 2 -overlay.load 5tt_coreg.mif -overlay.colourmap 1

echo "-- Step (18) Creating seed region along the GM/WM boundary."
	5tt2gmwmi 5tt_coreg.mif gmwmSeed_coreg.mif

	# 	To check the interface, do: mrview dwi_den_preproc_unbiased.mif -overlay.load gmwmSeed_coreg.mif
```	