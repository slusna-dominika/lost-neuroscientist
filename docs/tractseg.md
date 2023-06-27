## Run TractSeg using existing peaks from MRtrix ouput
TractSeg uses convolutional neural networks to perform bundle segmentation on the field of FODF peaks (principal directions). It does not create tractography per se, but it recently included tract orientation mapping to achieve bundle-specific tractography.

```bash
sh2peaks WM_FODs.nii.gz peaks.nii.gz -num 3

TractSeg -i peaks.nii.gz
TractSeg -i peaks.nii.gz --output_type endings_segmentation
TractSeg -i peaks.nii.gz --output_type TOM
Tracking -i peaks.nii.gz --tracking_dir TOM_trackings_filt
```

## Sampling of tensor-derived metrics (e.g., FA) along tracts
```bash
Tractometry -i /path/to/tractseg/tractseg_output/TOM_trackings_filt/ -o /path/to/tractseg/Tractomentry.csv -e /path/to/tractseg/tractseg_output/endings_segmentations/ -s /path/to/tractseg/FA_MNI.nii.gz
```

## Computing mean metric over bundle masks
```matlab
clear all; clc

% ------------------------------------------------------------------------
% User input

% Path to subject folder
subjects{1} = '/path/to/tractseg/subject/folder/sub01';
subjects{2} = '/path/to/tractseg/subject/folder/sub02';

% Tracts to estimate 
tract{1} = 'AF_left.nii';
tract{2} = 'AF_right.nii';
tract{3} = 'CST_left.nii';
tract{4} = 'CST_right.nii';

% Metrics to be estimated over tract masks
metric{1} = 'FA_MNI.nii';
metric{2} = 'MD_MNI.nii';
% ------------------------------------------------------------------------

% Mean metric (all subjects and tracts)
output = zeros(length(subjects),length(xtract),length(metric));

for i=1:length(subjects) %subjects

	for m=1:length(metric)
	path_to_metric = fullfile(subjects{i},metric{m});
	disp(path_to_metric)
	% ---- Load metric data
    V_metr  = spm_vol(path_to_metric);
    metr    = spm_read_vols(V_metr);

		for j=1:length(xtract)
		path_to_tract = fullfile(subjects{i}, 'tractseg_output/xtract/', tract{j});
		disp(path_to_tract)
		% ---- Load tract mask
        V_tract     = spm_vol(path_to_tract);
        tract_mask  = spm_read_vols(V_tract);
        indices     = (tract_mask>0.5);
		output(i,j,m) = mean(metr(indices));
		
		end
	end
end
```