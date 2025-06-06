# Geneva Stroke Imaging Preprocessing - *GSprep*
This repository hosts preprocessing tools for acute stroke imaging. 
It has been initially created for the Geneva Stroke Dataset (GSD). 

## How-to
### Installation

Run `python setup.py install`. 

This installs:
- *GSprep* : as set of preprocessing tools
- *gsd_pipeline* : the preprocessing pipeline used for the GSD

### Reference

If you use this work for your research, please cite this paper:

> Klug J, Dirren E, Preti MG, Machi P, Kleinschmidt A, Vargas MI et al. Integrating regional perfusion CT information to improve prediction of infarction after stroke. J Cereb Blood Flow Metab 2020: 0271678X20924549.

BibTex entry:

```bibtex
@article{klug2020integrating,
  title={Integrating regional perfusion CT information to improve prediction of infarction after stroke},
  author={Klug, Julian and Dirren, Elisabeth and Preti, Maria G and Machi, Paolo and Kleinschmidt, Andreas and Vargas, Maria I and Van De Ville, Dimitri and Carrera, Emmanuel},
  journal={Journal of Cerebral Blood Flow \& Metabolism},
  pages={0271678X20924549},
  year={2020},
  publisher={SAGE Publications Sage UK: London, England}
}
```

### GSD Pre-processing Pipeline
##### 1. Data Verification
- pre_verification/find_empty_folders.py : find empty folders in subject directories hinting towards failed exports and save in excel file
- pre_verification/verify_RAPID37.py : check that all subjects with perfusion CTs have 37 RAPID images (and not 11)
- utils/extract_unknown_studies_folder.py : extract images saved as an unspecified "study" folder
- utils/extract_RAPID37_folder.py : extract images saved as an unspecified "RAPID37" folder

##### 2. Data Extraction

0. image_name_config.py : file defining name space of relevant MRI and CT sequences
1. organise.py : organise into a new working directory, extracting only useful and renaming to something sensible + anonymize patient information
    - Nb.: watch out for patient seperator, patient might be missed if they use a different seperator in their file/folder names
2. verify_completeness.py (dcm) : verify that all necessary files are present
3. to_nii_batch.py : batch convert subject DICOMs to Nifti
4. utils/flatten.py : flatten into an MRI and a CT folder
5. verify_completeness.py (nifti) : verify that all necessary files are present

Verify clinical exclusion criteria:
- Time between CT and MRI (reported in anonymisation_key from organise.py)
- other exclusion criteria: IAT before CT, no treatment received
    - extract_patient_characteristics.py: extract relevant patient characteristics from main excel database

#### 3. CT Preprocessing
##### 3.1 General Pipeline
This is the general pipeline, for specificities for perfusion maps, Angio-CT or 4D perfusion CT, please read the sections below first.

--> Add this point data can be uploaded to a remote server
1. skull_strip/skull_strip_wrapper.py : batch skull strip native CTs of multiple patients and segment CSF
2. Unless Perfusion maps are to be excluded, follow the steps in 3.2
3. matlab_tools/perfCT_coregistration_wrapper.m : coregister perfusion CT to betted native CT
4. matlab_tools/perfCT_normalisation_wrapper.m : normalise perfusion CT and native CT to CT_MNI (if this needs to be reprocessed, step 2. also needs to be done again)

##### 3.2 Perfusion maps 
Map order: Tmax, CBF, MTT, CBV

1. utils/resolve_RAPID_4D_maps: resolve RAPID maps with 4 dimensions
    - get_RAPID_4D_list: find subjects with 4D RAPID maps
    - resolve_RAPID_4D_maps : reduce dimensions to 3D of given subjects (subjects may need to be downloaded from the server first as this function requires an Xserver for graphical feedback)
2. Proceed to the steps described in 3.1
3. Dataset post-processing (see 6.1): some maps need intensity rescaling

##### 3.3 Angio-CT
If Angio-CTs are used, the head has to be extracted from the half-body image, after which the skull has to be 
stripped from the image. Finally vessels are extracted. 

Angio Sequence used is Angio_CT_075_Bv40 as the contrast between contrast agent and brain matter is greater

0. Follow all the Data Verification and Extraction steps with the include_angio setting set to True (organise.py)
1. Do the skull_strip step mentioned above for the rest of the CT (3.1, step 1) --> CSF segmentation

2. angio_ct_extraction/brain_extract_wrapper.py : center FOV on head and extract brain only (applies the priorly obtained CSF mask)
3. angio_ct_extraction/vx_segmentation/wrapper_vx_extraction.py: extract only brain_vessels (hessian vesselness)

4. Do all the rest of CT processing from step 2 (#3.1) on with the with_angio option for the normalisation (see above)

5. Post-processing (see below)
- masking/brain_mask.py with restrict_to_RAPID_maps set to False
- Skip lesion_masking 

##### 3.4 4D Perfusion CT

0. Follow all the Data Verification and Extraction steps with the include_pCT setting set to True (organise.py)
1. pCT_preprocessing_pipeline: motion correction, coregistration and brain extraction of 4D pCT files
- if spm is not in matlab default path, it must be set with the spm_path argument
- If `ValueError: unknown locale: UTF-8` occurs, use `export LC_ALL=en_US.UTF-8` and  `export LANG=en_US.UTF-8` in your terminal
2. Follow all steps mentioned in the general CT processing (#3.1) with the with_pCT option on (see above)

3. Post-processing (see below)
- masking/brain_mask.py with restrict_to_RAPID_maps set to False
- Skip lesion_masking 

#### 4. MRI preprocessing

(0. matlab/dwi_mri_coregistration_wrapper.m : if DWI is used, it has to be coregistered to T2 first) 
1. matlab/mri_coreg_normalisation_wrapper.m : recenter subject CT, co-register T2 to subject CT, co-register T2 to CT-MNI and normalise to CT-MNI

#### 5. Post-processing
As RAPID performs excessive skull-stripping, same crop has to be applied to lesion maps to remove lesions without underlying input data. 
At the same time the CSF_mask is integrated into the non-brain mask.
- binarize_maks.py : binarize all masks (lesions and others) by applying a threshold from the maximum value (this is necessary as sometimes drawn lesions are 0-1 or 0-255 and during the normalisation values can be slightly altered)
- masking/brain_mask.py : create brain masks based on RAPID perfusion maps
    - restrict_to_RAPID_maps True only if only using RAPID maps
- masking/mask_lesions.py : apply brain masks to lesions
    - Skip if using 4D perfusion CT or AngioCT
- utils/preprocessing_verification : visual verification of preprocessing

#### 6. Dataset creation and processing

- Create dataset with data_loader

#### 6.1 Dataset post-processing

- tools/perfusion_maps_tools/rescale_outliers : Intensity rescaling a dataset of RAPID perfusion maps 


#### Additional steps for using HD images 

HD images are not warped to CT-MNI space, and can thus conserve the initial voxel space.

After running the whole CT and MRI preprocessing, run these steps again with the high_resolution setting
- skull_strip_wrapper: with high_resolution = True
- brain_mask: with high_resolution = True
- mask_lesions: with high_resolution = True  
- binarize_lesions: with high_resolution = True

Some images may need additional field of view adjustment, as image dimensions may not match.
- gsd_pipeline/utils/reset_img_size_and_fov.ipynb : tool and guide to adjust fov

#### Requirements

- matlab
- spm12 : http://www.fil.ion.ucl.ac.uk/spm/software/spm12/
- Clinical Toolbox for SPM (https://www.nitrc.org/projects/clinicaltbx/) [Has to be in the folder spm12/toolbox]
- dcm2niix : https://github.com/rordenlab/dcm2niix


## Explanation
### I. Data Collection

1. Stroke patients are identified based on eRecords
2. Imaging data is downloaded
3. Subject extraction [preprocessing/organise.py]
    - Subjects with pCT are identified
    - Subjects with pCT and subsequent MRI are included. MRI has to contain a T2 and a DWI sequence and is chosen if it contains a lesion label file OR if it is the MRI closest in time to the pCT.
    - Subjects are anonymised [sha256 of last_name + '^' + first_name + '^' + patient_birth_date] 
4. Subject data is converted to nifti building on the dcm2niix software [preprocessing/to_nii_batch.py]
5. Exclusion criteria are verified
    - CT to MRI time > 10d
    - No treatment received
    - Intra-arterial treatment before CT
    - MRI imaging before CT
    - Major anatomic deformations visualised on MRI: massive edema with herniation, craniectomy, massive hemorrhagic transformation 

### II. Pre-processing
#### CT Pre-processing

1. Verify dimensions and reduce to 3D. Some pCT files are saved in a 4D format (mostly containing twice the same image). 
2. Brain-extraction [preprocessing/skull_strip/skull_strip_wrapper.py]
    - Skull-stripping as proposed by Muschelli et al. [^1] 
        - Images are threshold using a 0 – 100 Hounsfield units (HU) range.
        - Data were smoothed using a 3-dimensional Gaussian kernel (σ = 1mm3) and re-threshold to 0 – 100 HU. 
        - BET was applied using a 0.01 fractional intensity (FI) thresholds.
        - Any holes in the brain mask were filled.
    - CSF segmentation (inspired by the work by Manniesing et al. [^2])
        - Mean and standard deviation were determined of the intensity histogram within [0,400] HU. 
        - An upper intensity threshold μ − 1.96σ was applied to segment CSF.
        - Followed by erosion with a structuring element with 1 voxel radius to remove potential false positive voxels due to noise, and connected region growing using the same thresholds. 
        - The CSF mask was subjected to a morphological closing operation with a structuring element with 2 voxel radius.
3. Co-register perfusions CT [pCT] sequences to skull-stripped native CT sequences (as implemented by SPM12) [matlab/perfCT_coregistration_wrapper.m]
    - cost function: normalized mutual information
4. Normalize non-betted native CT to the CT-MNI template as published by Rorden et al [^3] and apply this transformation to the pCT sequences obtained from the prior step. [matlab/perfCT_normalisation_wrapper.m]   
    - Align position and orientation to CT-MNI template (Clinical toolbox for SPM: clinical_setorigin)
    - Apply normalisation (as implemented by SPM12) with parameters derived from the Clinical toolbox for SPM

![CT preprocessing pipeline](illustrations/CT_preprocessing.png?raw=true "CT preprocessing pipeline")

[^1]: Muschelli J, Ullman NL, Mould WA, Vespa P, Hanley DF, Crainiceanu CM. Validated automatic brain extraction of head CT images. Neuroimage. 2015 Jul 1;114:379–85.

[^2]: Manniesing R, Oei MTH, Oostveen LJ, Melendez J, Smit EJ, Platel B, et al. White Matter and Gray Matter Segmentation in 4D Computed Tomography. Scientific Reports. 2017 Mar 9;7(1):119. 

[^3]: Rorden C, Bonilha L, Fridriksson J, Bender B, Karnath H-O. Age-specific CT and MRI templates for spatial normalization. Neuroimage. 2012;61(4):957-65.

##### Optional Angio-CT Pre-Processing

Angio Sequence used is Angio_CT_075_Bv40 as the contrast between contrast agent and brain matter is greater

1. Brain-extraction [angio_ct_extraction/brain_extract_wrapper.py]
    - Use robustfov (FSL) to get FOV on head only
    - Use FLIRT with mutual Information (FSL) to coregister to SPC
    - Create Mask by skull stripping the SPC
    - Apply mask to angio
    
2. Blood vessel segmentation [angio_ct_extraction/wrapper_vx_extraction.py]
    - Threshold masked image at 90 HU
    
3. Co-registration and Normalization (see above)

#### MRI Pre-processing

1. Align position and orientation of the native non-betted CT to the CT-MNI template (derived from Clinical toolbox for SPM: clinical_setorigin) [matlab/mri_coreg_normalisation_wrapper.m]
2. Co-register T2-MRI sequences to non skull-stripped native CT sequences (as implemented by SPM12) and apply this transformation to the lesion labels [matlab/mri_coreg_normalisation_wrapper.m]
    - cost function: normalized mutual information
3. Align and co-register the native non-betted CT to the CT-MNI template and apply to the co-registered T2-MRI and lesion label obtained from above (derived from Clinical toolbox for SPM: clinical_setorigin) 
4. Normalize non-betted native CT to the CT-MNI template and apply this transformation to the T2-MRI and lesion label obtained from the prior step. [matlab/mri_coreg_normalisation_wrapper.m]   
    - Apply normalisation (as implemented by SPM12) with parameters derived from the Clinical toolbox for SPM

![MRI preprocessing pipeline](illustrations/MRI_preprocessing.png?raw=true "MRI preprocessing pipeline")

#### Lesion label sanitization

 Binarize all lesions by applying a 0.8 threshold from the maximum value (this is necessary as sometimes drawn lesions are 0-1 or 0-255 and during the normalisation values can be slightly altered) [/preprocessing/binarize_lesions.py]

#### Visual verification

All subject images are visualised to check if the lesion label, the pCT maps, the native CT sequence and the T2 MRI are aligned correctly. Although this is rare, some subjects have to be excluded because of a failure in the prior preprocessing. This is mainly due to a native CT sequence of reduced quality (subject movement, not all skull scanned, external devices) as they are obtained in an emergency setting.

#### Masking

As RAPID performs excessive skull-stripping, some parts of the brain are not covered by the pCT maps.   
1. A pCT covered brain map is created for each subject [masking/brain_mask.py]
    - pCT coverage is defined if at least 3 of the four pCT are not null
    - subsequently brain area labeled as CSF by the segmentation defined above is substracted
2. The obtained brain mask is applied to the lesion labels to avoid having lesion data where no pCT data is defined

The obtained brain mask is applied to the train and test sets throughout the cross-validation, as the only areas of the brain space that are of interest for this analysis are the ones where pCT maps are defined.
