In the following guide, we will go through the following steps:

1. [Anatomically constrained tractography](#anatomically-constrained-tractography)
2. [Generation of the SC matrix](#generation-of-the-sc-matrix)
3. [Extracting tract streamlines](#extracting-tract-streamlines)
4. [Generation of thesor-derived metrics (FA, MD, RD, AD)](#generation-of-thesor-derived-metrics-fa-md-rd-ad)


## Anatomically constrained tractography
```bash
echo "-- Step (1) Creating streamlines. The number of streamlines is up to debate. MRtrix recommends 100 M. Here we use 10 M."
	tckgen -act 5tt_coreg.mif -backtrack -seed_gmwmi gmwmSeed_coreg.mif -nthreads 8 -maxlength 250 -cutoff 0.06 -select 10000000 wmfod_norm.mif tracks_10M.tck

echo "-- Step (2) Creating a subset of streamlines for ease of visualization (as the original number is too heavy for your RAM)."
		tckedit tracks_10M.tck -number 200k smallerTracks_200k.tck
		tckedit tracks_10M.tck -number 10k superSmall_10k.tck

	# 	Visualize via: mrview dwi_den_preproc_unbiased.mif -tractography.load smallerTracks_200k.tck

echo "-- Step (3) Refining the streamlines via SIFT2 (1M, 200k) informed by spherical deconvolution where streamline densities are proportional to fiber densities. This tractogram represent the sum of streamline weights not a mere number of streamline."
	tcksift2 -act 5tt_coreg.mif -out_mu sift_mu.txt -out_coeffs sift_coeffs.txt -nthreads 8 tracks_10M.tck wmfod_norm.mif sift_1M.txt

echo "-- Step (4) Coregister the raw T1."
	mrtransform T1.mif –linear diff2struct_mrtrix.txt –inverse T1_coreg.mif
```

## Generation of the SC matrix
```bash
echo "-- Step (5) Run recon-all"
    recon-all -i T1.nii -s T1_recon -all

echo "--Step (6) Create parcellation for the MRtrix output based on Freesurfer default atlas"
    labelconvert T1_recon/mri/aparc+aseg.mgz $FREESURFER_HOME/FreeSurferColorLUT.txt /usr/local/mrtrix3/share/mrtrix3/labelconvert/fs_default.txt parcels.mif

echo "--Step (8) Register atlas-based volumetric parcellation to diffusion space"
    mrtransform parcels.mif –linear diff2struct_mrtrix.txt –inverse –datatype uint32 parcels_coreg.mif

echo "-- Step (9) Create a whole-brain matrix representing the streamlines between each parcellation pair in the atlas. Scale it by the volume of atlas region."
	tck2connectome -symmetric \ # Make matrix symmetric 
	-zero_diagonal \ # Set diagonal line to 0
	-scale_invnodevol \ # Scale by volume (other options: scale by streamline length, inverse streamline length or vector values)
	-tck_weights_in sift_1M.txt \ # Specify a text scalar file containing the streamline weights
	tracks_10M.tck \ # The input track file
	parcels.mif \ # The input node parcellation image
	parcels.csv \ # Output file containing edge weights
	-out_assignment assignments_parcels.csv / # Output the node assignments of each streamline to a file (later used in connectome2tck for extracting pathways of interest)
```

## Extracting tract streamlines
**Extract streamlines between pair atlas regions**
```bash
connectome2tck –nodes 23,72 \ # Modify the number according to assigned numbers of atlas regions in fs_default.txt
–exclusive \ # Only select tracks that exclusively connect nodes from within the list of nodes of interest
-tck_weights_in sift_1M.txt \
tracks_10M.tck \ # The input track file
assignments_parcels.csv \ # Input text file containing the node assignments for each streamline
sift_1M.txt \ # Specify a text scalar file containing the streamline weights
motor / # The output file name / prefix
```
**Extract the streamlines connecting node X to all other nodes in the parcellation, with one track file for each edge**
```bash
connectome2tck -nodes 15 \ # Modify the number according to assigned numbers of atlas regions in fs_default.txt
tracks_10M.tck \ # The input track file
-tck_weights_in sift_1M.txt \
assignments_parcels.csv \ # Input text file containing the node assignments for each streamline
sift_1M.txt \ # Specify a text scalar file containing the streamline weights
–files per_node \ # Resulting files should be one file for each region/node that we analyze
M1_to_brain / # The output file name / prefix
```
**Filter track-file streamlines based on an ROI coordinates**
```bash
tckedit –include 5.26,-2.48,-39.4,3 \ # Specify the X,Y,Z coordinates and the radius of the sphere your ROI should have
tracks_10M.tck \ # The input track file 
-tck_weights_in sift_1M.txt \
mytract.tck / # Modify the output file name
```
**Filter track-file streamlines based on an multiple ROI coordinates or masks**
```bash
tckedit tracks_10M.tck mytract.tck \
-include ROI1.mif -include 5.26,-2.48,-39.4,3 # options: exclude, mask, include ordered
-tck_weights_in sift_1M.txt \
-minlength 25 / # minimum length of any streamline in mm (also maxlength)
```

## Generation of tensor-derived metrics (FA, MD, RD, AD)
```bash
echo "-- Step (1) Create a tensor image from DWI data to later compute tensor-direved parameters such as FA"
    dwi2tensor dwi_den_preproc_unbiased.mif \ # Input DWI image 
    dt_dwi_den_preproc_unbiased.mif \  # The output file name / prefix
    -fslgrad dwi.bvec dwi.bval 

echo "-- Step (2) Compute tensor-direved parameters of the brain"
    tensor2metric -fa fa_dwi.mif \
	-mask mask.mif \ # Mask with the binary brain mask
	dt_dwi_den_preproc_unbiased.mif /

echo "-- Step (3) Sample tensor-derived paramaters along the streamlines (mean value per streamline)"
	tcksample myTract.tck fa_dwi.mif output -stat_tck mean
```

