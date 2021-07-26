![](./pnl-bwh-hms.png)

[![DOI](https://zenodo.org/badge/doi/10.5281/zenodo.3258854.svg)](https://doi.org/10.5281/zenodo.3258854) [![Python](https://img.shields.io/badge/Python-3.6-green.svg)]() [![Platform](https://img.shields.io/badge/Platform-linux--64%20%7C%20osx--64-orange.svg)]()

Developed by Tashrif Billah, Sylvain Bouix, and Yogesh Rathi, Brigham and Women's Hospital (Harvard Medical School).


Table of Contents
=================

   * [Table of Contents](#table-of-contents)
   * [Citation](#citation)
   * [Pipeline scripts overview](#pipeline-scripts-overview)
   * [Pipeline graphs](#pipeline-graphs)
      * [Structural](#structural)
      * [Diffusion](#diffusion)
         * [FslEddy PnlEpi or PnlEddy PnlEpi](#fsleddypnlepi-or-pnleddypnlepi)
         * [PnlEddy or FslEddy](#pnleddy-or-fsleddy)
         * [TopupEddy](#topupeddy)
      * [Tractography](#tractography)
   * [DICOM to NIFTI](#dicom-to-nifti)
   * [Temporary directory](#temporary-directory)
   * [Axis alignment](#axis-alignment)
   * [Resampling](#resampling)
   * [Gibbs unringing](#gibbs-unringing)
   * [Masking](#masking)
      * [1. Structural mask](#1-structural-mask)
         * [i. Mask from training data](#i-mask-from-training-data)
         * [ii. Mask using FSL Bet](#ii-mask-using-fsl-bet)
         * [ii. Mask through registration](#ii-mask-through-registration)
      * [2. Diffusion mask](#2-diffusion-mask)
         * [i. Baseline image](#i-baseline-image)
         * [ii. FSL Bet](#ii-fsl-bet)
      * [3. Multiply by mask](#3-multiply-by-mask)
   * [Mask filtering](#mask-filtering)
   * [Eddy correction](#eddy-correction)
      * [i. Through registration](#i-through-registration)
      * [ii. Using FSL eddy](#ii-using-fsl-eddy)
   * [Epi correction](#epi-correction)
      * [i. Through registration](#i-through-registration-1)
      * [ii. Using FSL topup and eddy](#ii-using-fsl-topup-and-eddy)
   * [UKFTractography](#ukftractography)
   * [FreeSurfer](#freesurfer)
   * [FreeSurfer segmentation in T1w space](#freesurfer-segmentation-in-t1w-space)
   * [FreeSurfer segmentation in DWI space](#freesurfer-segmentation-in-dwi-space)
      * [i. Direct registration](#i-direct-registration)
      * [ii. Through T2 registration](#ii-through-t2-registration)
   * [White matter query](#white-matter-query)
   * [Render white matter tracts](#render-white-matter-tracts)
   * [Extract measures from tracts](#extract-measures-from-tracts)
   * [800 cluster white matter percellation](#800-cluster-white-matter-percellation)
   * [Additional scripts](#additional-scripts)


Table of Contents created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)



# Citation

If this pipeline is useful in your research, please cite as below:

Billah, Tashrif; Bouix, Sylvain; Rathi, Yogesh; NIFTI MRI processing pipeline,
https://github.com/pnlbwh/pnlNipype, 2019, DOI: 10.5281/zenodo.3258854



**NOTE** All trivial *Meta-switches* and *Hidden-switches* have been omitted from help messages in this tutorial.


# Pipeline scripts overview

Please see [Pipeline scripts overview](README.md#pipeline-scripts-overview).


# Pipeline graphs

## Structural

![](FreesurferN4Bias.png)

Use only the left branch when no T2w is available

## Diffusion

### FslEddy+PnlEpi or PnlEddy+PnlEpi

![](FslEddyEpi.png)


### PnlEddy or FslEddy

Upto *FslEddy/PnlEddy* of the left branch in the above

### TopupEddy

Can be done when two opposing acquisitions--AP and PA are available

![](TopupEddy.png)


## Tractography

![](Fs2Dwi.png)




# DICOM to NIFTI

*pnlNipye* accepts 3D/4D MRI in NIFTI format. To generate NIFTI from DWI directly, use [dcm2niix](https://github.com/rordenlab/dcm2niix)

    dcm2niix -o outputDir -f namePrefix -z y dicomDir


# Temporary directory

Both *pnlpipe* and *pnlNipype* have centralized control over various temporary directories created down the pipeline. 
The temporary directories can be large, and may possibly clog the default `/tmp/` directory. You may define custom 
temporary directory with environment variable `PNLPIPE_TMPDIR`:

    mkdir ~/tmp/
    export PNLPIPE_TMPDIR=~/tmp/

# Axis alignment

Once you have the NIFTI file (and associated bval/bvec files), it is a good practice to axis align and center the MRI. 
Axis alignment ensures there is non-zero non-diagonal element in the affine transform of the NIFTI image. On the other 
hand, centering ensures the origin is set at half the image size across all axes adjusted for resolution.


There may be a neuroimaging software that fails if oblique/shifted MRI is provided. So, axis alignment and centering is 
important. You can use the following script for this purpose:

> nifti_align -h

    Axis alignment and centering of a 3D/4D NIFTI image
    
    Usage:
        nifti_align [SWITCHES] 
    
    Switches:
        --axisAlign                         turn on for axis alignment
        --bvals VALUE:ExistingFile          bval file
        --bvecs VALUE:ExistingFile          bvec file
        --center                            turn on for centering
        -i, --input VALUE:ExistingFile      a 3d or 4d nifti image; required
        -o, --outPrefix VALUE:str           prefix for naming dwi, bval, and bvec files


Example usage:

    nifti_align -i dwiNifti --bvals bvalFile --bvecs bvecFile -o nifti-xc
    nifti_align -i t1Nifti --center
    nifti_align -i t2Nifti --axisAlign
    
    
You can choose to either `axisAlign` or `center` or both. The first example will do both by default. If you 
don't provide a `--outPrefix` (or `-o`) as shown in other examples, it will be inferred from the input file name. You 
must provide bval and bvec with diffusion weighted 4D NIFTI.


**NOTE** Axis alignment involves rotation of the image. In this case, bvecs will also be rotated. However, bvals remain 
untouched throughout the axis alignment and centering steps.

You can see the affine transform as follows (launch a python interpreter using `ipython`:

    
    $ ipython 

    Python 3.6.7 | packaged by conda-forge | (default, Feb 28 2019, 09:07:38) 
    Type 'copyright', 'credits' or 'license' for more information
    IPython 7.4.0 -- An enhanced Interactive Python. Type '?' for help.

    In [1]: import nibabel as nib                                                                                     
    
    In [2]: img= nib.load('niftiFile.nii.gz')                                                      
    
    In [3]: img.affine                                                                                                
    Out[3]: 
    array([[ -1.5       ,   0.        ,   0.        , 120.        ],
           [  0.        ,   1.47143269,  -0.29135156, -96.01908875],
           [  0.        ,   0.29135156,   1.47143269, -63.87701035],
           [  0.        ,   0.        ,   0.        ,   1.        ]])
           
    
    In [4]: img.header['dim']                                                                                         
    Out[4]: array([  3, 160, 160,  93,   1,   1,   1,   1], dtype=int16)
    
    In [5]: img.header['pixdim']                                                                                      
    Out[5]: array([-1. ,  1.5,  1.5,  1.5,  1. ,  0. ,  0. ,  0. ], dtype=float32)



The above NIFTI image is neither axis aligned (has non-zero off-diagonal elements) 
nor centered (fourth column is not (size-1)/2*resolution). 
After application of the above script, the affine transform should look like the following:

    In [5]: img= nib.load('niftiFile-xc.nii.gz')                                                      
    
    In [6]: img.affine                                                                                                
    Out[3]:
    array([[  -1.5 ,    0.  ,    0.  , -119.25],
           [   0.  ,    1.5 ,    0.  ,  119.25],
           [   0.  ,    0.  ,    1.5 ,   69.  ],
           [   0.  ,    0.  ,    0.  ,    1.  ]])



# Resampling

`ResampleImage` executable from ANTs is used for resampling an image to a desired size/resolution. 
Resampling is done at 3D level. If an image is 4D, it is split into 3D volumes, resampled, and then merged back.


> resample -h

    usage: resample.py [-h] [-i INPUT] [-o OUTPREFIX] [--ncpu NCPU] [--size SIZE]
                       [--order ORDER]
    
    Resample an MRI using ANTs ResampleImage executable. If the image is 4D, it is
    split to 3D along the last axis, resampled at 3D level, and merged back.
    
    optional arguments:
      -h, --help            show this help message and exit
      -i INPUT, --input INPUT
                            input3D/4D MRI
      -o OUTPREFIX, --outPrefix OUTPREFIX
                            resampled image and corresponding bval/bvec are saved
                            with outPrefix
      --ncpu NCPU           default 4, you can increase it at the expense of RAM
      --size SIZE           resample to MxNxO size or resolution, if all of
                            M,N,O<5, it is interpreted as resolution
      --order ORDER         For details about order of interpolation, see
                            ResampleImage --help, the default for masks is 0
                            (nearest neighbor) while for all other images it is 4
                            (Bspline [order=5])


Example usage:
    
    resample dwiNifti dwiReNifti 8
    resample t1Nifti t1ReNifti --order 1



# Gibbs unringing

This pipeline uses [DIPY implementation](https://dipy.org/documentation/1.1.1./examples_built/denoise_gibbs/#example-denoise-gibbs) of Gibbs unringing algorithm.

> unring -h

    Usage:
    unring <dwi> <outPrefix> <ncpu>
    Gibbs unringing of all DWI gradients using DIPY
    Default ncpu=4, you can increase it at the expense of RAM


Example usage:
    
    unring dwiNifti dwiUnNifti 8
    
    

# Masking

Masking refers to skull stripping. Neuroimage analysis algorithms have been found to perform better when skull is stripped. 
Therefore, masking is an important step. During masking, you want to ensure most of the brain is included in the 
mask while non brain is excluded.


## 1. Structural mask

T1/T2 images are structural images. Structural MRI mask can be created in two ways:


### i. Mask from training data

A set of [T1 training images](https://github.com/pnlbwh/trainingDataT1AHCC) and [T2 training images](https://github.com/pnlbwh/trainingDataT2Masks) are given to the following script. 
The script non linearly registers each training image to the target T1/T2 image. Transforms obtained through registration 
are applied on the corresponding training masks (or labelmaps). Thereby, a set of candidate masks (or labelmaps) are obtained in the 
space of the target T1/T2 image. The set of candidate masks (or labelmaps) are fusioned to create final mask (or labelmap).


> nifti_atlas --help-all

    Makes atlas image/labelmap pairs for a target image.
    Option to merge labelmaps via averaging or AntsJointFusion.
    Specify training images and labelmaps via a csv file.
    Put the images with any header in the first column,
    and labelmaps with proper headers in the consecutive columns.
    The headers in the labelmap columns will be used to name the generated atlas labelmaps.
    
    Usage:
        nifti_atlas [SWITCHES]
        
    Switches:
        -d                                               Debug mode, saves intermediate labelmaps to atlas-debug-<pid> in output directory
        --fusion VALUE:{avg, wavg, antsJointFusion}      Also create predicted labelmap(s) by combining the atlas labelmaps: avg is naive mathematical
                                                         average, wavg is weighted average where weights are computed from MI between the warped
                                                         atlases and target image, antsJointFusion is local weighted averaging; the default is wavg
        -n, --nproc VALUE:str                            number of processes/threads to use (-1 for all available); the default is 4
        -o, --outPrefix VALUE:str                        output prefix, output labelmaps are saved as outPrefix_mask.nii.gz, outPrefix_cingr.nii.gz,
                                                         ...; required
        -t, --target VALUE:ExistingFile                  target image; required
        --train VALUE:str                                --train t1; --train t2; --train trainingImages.csv; see pnlNipype/docs/TUTORIAL.md to know
                                                         what each value means




Example usage:
    
    nifti_atlas -t t1Nifti -o /tmp/T1-Mabs -n 8 --train t1
    nifti_atlas -t t2Nifti -o /tmp/T2-Mabs -n 8 --train t2
    nifti_atlas -t t1Nifti -o /tmp/T1-Mabs -n 8 --train ~/pnlpipe/soft_dir/trainingDataT1AHCC-d6e5990/trainingDataT1Masks-hdr.csv

As shown in the above, pre-packaged training data can be specified with just `--train t1` or `--train t2` without having to provide path to csv file. 
They are equivalent to the following:

`--train t1`:
> $PNLPIPE_SOFT/trainingDataT1AHCC-*/trainingDataT1Masks-hdr.csv

`--train t2`:
> $PNLPIPE_SOFT/trainingDataT2Masks-*/trainingDataT2Masks-hdr.csv

**NOTE** For using shortcuts `t1` and `t2`, make sure to define the environment variable [`PNLPIPE_SOFT`](README.md#pnlpipe-software). Otherwise, provide path to csv file
    
The csv file used here is `trainingDataT1AHCC-d6e5990/trainingDataT1AHCC-hdr.csv` which can be generated by running 
[mktrainingcsv.sh](https://github.com/pnlbwh/trainingDataT1AHCC/blob/master/mktrainingcsv.sh). However, it has been already generated for you when you installed the pipeline.

Finally, `-n 8` specifies the number of processors you can use for this purpose. See [Multiprocessing](README.md/#multiprocessing) to learn more about it.




### ii. Mask using FSL Bet

At the very least, you can use FSL bet to create the mask. However, you should visually look at the mask for correctness 
and edit it if necessary. The following is a generalized script for creating FSL bet mask from 3D/4D images.

> nifti_bet_mask -h

    Extracts the brain mask of a 3D/4D nifti image using fsl bet command
    
    Usage:
        nifti_bet_mask [SWITCHES] 
    
    Switches:
        --bvals VALUE:ExistingFile           bval file for 4D DWI, default: inputPrefix.bval
        -f VALUE:str                        threshold for fsl bet mask; the default is 0.25
        -i, --input VALUE:ExistingFile      input 3D/4D nifti image; required
        -o, --outPrefix VALUE:str           prefix for output brain mask (default: input prefix), output file is named
                                            as prefix_mask.nii.gz


Example usage:

    nifti_bet_mask -i dwiNifti --bvals bvalFile
    nifti_bet_mask -i t1Nifti -o t1bet

For 4D diffusion weighted image, bet mask is created from first B0 image image. For 3D structural image, 
it is created directly.


### ii. Mask through registration

Sometimes, you have a batch of structural images (T1/T2) for all of which you do not want to create (computationally 
expensive mask)[#i-mask-from-training-data]. In that case, you can create mask from training data for only one subject 
and rigidly register that mask to the space of other subjects. The following script emulates this function:

> nifti_makeRigidMask -h

    Align a given labelmap (usually a mask) to make another labelmap
    
    Usage:
        nifti_makeAlignedMask [SWITCHES]

    Switches:
        -i, --input VALUE:ExistingFile         structural (nrrd/nii); required
        -l, --labelmap VALUE:ExistingFile      structural labelmap, usually a mask (nrrd/nii); required
        -o, --output VALUE:str                 output labelmap (nrrd/nii); required
        --reg VALUE:{rigid, SyN}               ANTs registration method: rigid or SyN; the default is rigid
        -t, --target VALUE:ExistingFile        target image (nrrd/nii); required


Example usage:
    
    nifti_makeAlignedMask -i oneT1 -l atlasMaskForOneT1 -t otherT1 -o atlasMaskForOtherT1
    
In the above script, the `oneT1` is rigidly registered to the space of `otherT1`. Transform obtained through this 
registration is applied upon `atlasMaskForOneT1` to create `atlasMaskForOtherT1`.


## 2. Diffusion mask

Diffusion masking is trickier than structural masking since the former has multiple volumes stacked one after another. 
To create a diffusion mask, the volumes should be motion corrected such that one single mask encompasses brain regions 
across all volumes.

### i. Baseline image

Since there are multiple volumes in a diffusion weighted image, you want to extract the baseline image corresponding to 
zero Bvalue that may be used to create a mask for the DWI. The following script by default extracts the first volume that 
has Bvalue less than a threshold. Alternatively, you can choose to have `--all`, `--avg`, and `--min` baseline images. 
See the following help message to know more about those options.

> nifti_bse -h

    Extracts the baseline (b0) from a nifti DWI. Assumes
    the diffusion volumes are indexed by the last axis. Chooses the first b0 as the
    baseline image by default, with option to specify one.
    
    Usage:
        nifti_bse [SWITCHES]
    
    Switches:
        --all                               turn on this flag to choose all bvalue<threshold volumes as the baseline
                                            image, this is an useful option if you want to feed B0 into
                                            topup/eddy_openmp
        --avg                               turn on this flag to choose the average of all bvalue<threshold volumes as
                                            the baseline image, you might want to use this only when eddy/motion
                                            correction has been done before
        --bvals VALUE:ExistingFile           bval file, default: dwiPrefix.bval
        -i, --input VALUE:ExistingFile      DWI in nifti format; required
        -m, --mask VALUE:ExistingFile       mask of the DWI in nifti format; if mask is provided, then baseline image
                                            is masked
        --min                               turn on this flag to choose minimum bvalue volume as the baseline image
        -o, --output VALUE:str              extracted baseline image (default: inPrefix_bse.nii.gz)
        -t, --threshold VALUE:str           threshold for b0; the default is 45.0
        

Example usage:

    nifti_bse -i dwiNifti --bvals bvalFile -o bseNifti
    nifti_bse -i dwiNifti --bvals bvalFile -o bseNifti --avg
    

### ii. FSL Bet

Once you have the baseline image which is a 3D NIFTI, FSL bet mask creation is same as [structural BET masking](#-ii-mask-using-fsl-bet)



## 3. Multiply by mask

> masking -h

    Multiplies an image by its mask
    
    Usage:
        masking [SWITCHES] 
    
    Switches:
        -d, --dimension VALUE:str           Input image dimension: 3/4; required
        -i, --input VALUE:ExistingFile      nifti image, 3D/4D image; required
        -m, --mask VALUE:ExistingFile       nifti mask, 3D image; required
        -o, --output VALUE:str              Masked input image; required


The above script basically uses `ImageMath` from *ANTs* to multiply an image by its mask:

    ImageMath 4 dwiMaskedNifti m dwiNifti maskNifti
    ImageMath 3 t1MaskedNifti m t1Nifti maskNifti
    
You can also use the following command from *FSL* to mask an image:

    fslmaths volNifti -mas maskNifti maskedVolNifti
    
The advantage of the latter command is, you wouldn't have to provide dimension of the image you want to mask.



# Mask filtering

Often it is necessary to perform morphological operation--erosion/dilation on a mask to get rid of holes and islands. 
We provide a script for doing that. The operation also smooths out the boundary of the mask. We have used a 
six-connected 3D structural element for that.

> maskfilter -h

    This python executable replicates the functionality of
    https://github.com/MRtrix3/mrtrix3/blob/master/core/filter/mask_clean.h
    It performs a few erosion and dilation to remove islands of non-brain region in a brain mask.
    
    Usage: maskfilter input scale output
    
    See https://github.com/MRtrix3/mrtrix3/blob/master/core/filter/mask_clean.h for details

    

# Eddy correction


## i. Through registration

This is the conventional PNL way of Eddy correction. It registers each volume to the baseline image. It also 
applies the transform to appropriately rotate the corresponding bvecs.

> pnl_eddy -h

    Eddy current correction.
    
    Usage:
        pnl_eddy [SWITCHES]
    
    Switches:
        --bvals VALUE:ExistingFile      bval file for DWI; required
        --bvecs VALUE:ExistingFile      bvec file for DWI; required
        -d                             Debug, saves registrations to eddy-debug-<pid>
        --force                        Force overwrite
        -i VALUE:ExistingFile          DWI in nifti; required
        -n, --nproc VALUE:str          number of threads to use, if other processes in your computer
                                       becomes sluggish/you run into memory error, reduce --nproc; the
                                       default is 4
        -o VALUE:str                   Prefix for eddy corrected DWI; required
        

Example usage:
    
    pnl_eddy -i dwiNifti --bvals bvalFile --bvecs bvecFile -o dwiNifti-Ed
        

## ii. Using FSL eddy

FSL *eddy_openmp* is the new way of Eddy correction. Running *eddy* is a little bit complicated and computationally 
expensive than the *pnl_eddy*. The former attempts to combine the correction for susceptibility and 
eddy currents/movements so that there is only one single resampling. *eddy_openmp* attempts to model the diffusion signal. 
So, we need to inform eddy of the diffusion direction/weighting that was used for each volume. It can  can utilise the 
information from different acquisitions that modulate how off-resonance translates into distortions. An example of this 
would be acquisitions with different polarity of the phase-encoding. Hence, we also need to inform eddy about 
how each volume was acquired. The above information is provided through `--acqp`, a *.txt* file. On the other hand, 
`--index` file provides the mapping of acquisition parameter to each volume. See [FSL wiki](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddy/UsersGuide) 
to learn more about Eddy correction.



> fsl_eddy -h

    
    Eddy correction using eddy_openmp/cuda command in fsl
    For more info, see https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddy/UsersGuide
    You can also view the help message typing:
    `eddy_openmp` or `eddy_cuda` 
    
    Usage:
        fsl_eddy [SWITCHES] 
    
    Switches:
        --acqp VALUE:ExistingFile        acquisition parameters file (.txt); required
        --bvals VALUE:ExistingFile       bvals file of the DWI); required
        --bvecs VALUE:ExistingFile       bvecs file of the DWI); required
        --config VALUE:ExistingFile      config file for FSL eddy tools; see scripts/eddy_config.txt; 
                                         copy this file to your directory, edit relevant sections, 
                                         and provide as --config /path/to/my/eddy_config.txt; required
        --dwi VALUE:ExistingFile         nifti DWI image); required
        --eddy-cuda                      use eddy_cuda instead of eddy_openmp, requires
                                         fsl/bin/eddy_cuda and nvcc in PATH
        -f VALUE:str                     threshold for fsl bet mask; the default is 0.25
        --index VALUE:ExistingFile       mapping file (.txt) for each gradient --> acquisition
                                         parameters; required
        --mask VALUE:ExistingFile        mask for the DWI; if not provided, a mask is created using
                                         fsl bet
        --out VALUE:NonexistentPath      output directory; required


Example usage:

Let's say we have a 4D nifti file with 20 gradients directions. First 10 volumes were acquired in anterior-->posterior (AP) 
direction while the rest were acquired in (PA) posterior-->anterior (PA) direction. Then, `acqparams.txt` should contain 
two lines:

    0 -1 0 0.050
    0  1 0 0.050
    
Details of this file can be found [here](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddy/UsersGuide)

Again, since first 10 volumes corresponds to 1st line while the last 10 corresponds to 2nd line, the `index.txt` file 
should look like the following:

    1 1 1 1 1 2 2 2 2 2
    
Then, you can run *fsl_eddy* as follows:

    fsl_eddy --dwi dwiNifti --mask maskNifti --bvals bvalFile --bvecs bvecFile --acqp acqparams.txt --index index.txt --out /tmp/Eddy/ --config /path/to/my/eddy_config.txt



**NOTE** Any additional arguments to *eddy_openmp*, *topup*, and *applytopup* can be provided by a copy of `scripts/eddy_config.txt` 
file.


**NOTE** We observed that `--repol` flag for outlier replacement does not work well for lower b-value shells. If this flag 
is provided, we run eddy_openmp/cuda twice-- *with* and *without* `--repol` and then replace *with* b-value<=500 shells 
by *without*.

![](./diff_repol.png)

**NOTE** You can tell the script to use `eddy_cuda` by using the flag `--eddy-cuda`. If CUDA binary `nvcc` is present 
in the environment variable PATH, then `eddy_cuda` is used. By default `eddy_cuda` runs on GPU #0 by default. 
In addition, you need to create a soft link `eddy_cuda` in `fsl/bin/` directory for 
`eddy_cuda8.0` or `eddy_cuda9.1` depending on the version of CUDA you have. If you want to override the default 
GPU device, use `export CUDA_VISIBLE_DEVICES=1` or whatever. However, do not get confused by the console message:


    ...................Allocated GPU # 0...................
    Loading prediction maker
    Evaluating prediction maker model
    Checking for outliers
    Calculating parameter updates



It will always show `GPU # 0`, but `nvidia-smi` command will show the actual GPU being used.




# Epi correction


## i. Through registration

If you have a T2 image acquired in the space of DWI, then you can run Epi correction in the conventional PNL way. 
In this method, T2 image is registered to the baseline image and the obtained transform is applied to the 
diffusion weighted volumes. 
 

> pnl_epi -h

    Epi distortion correction.
    
    Usage:
        pnl_epi [SWITCHES] 

    Switches:
        --bse VALUE:ExistingFile          b0 of the DWI
        --bvals VALUE:ExistingFile        bvals file of the DWI; required
        --bvecs VALUE:ExistingFile        bvecs file of the DWI; required
        -d, --debug                       Debug, save intermediate files in 'epidebug-<pid>'
        --dwi VALUE:ExistingFile          DWI; required
        --dwimask VALUE:ExistingFile      DWI mask; required
        --force                           Force overwrite if output already exists
        -n, --nproc VALUE:str             number of threads to use, if other processes in your computer becomes 
                                          sluggish/you run into memory error, reduce --nproc; the default is 4
        -o, --output VALUE:str            Prefix for EPI corrected DWI, same prefix is used for saving 
                                          bval, bvec, and mask; required
        --t2 VALUE:ExistingFile           T2w; required
        --t2mask VALUE:ExistingFile       T2w mask; required
        

Example usage:

    pnl_epi --dwi dwiNifti --dwimask maskNifti --t2 t2Nifti --t2mask t2MaskNifti -o dwiEpiPrefix
    pnl_epi --dwi dwiNifti --dwimask maskNifti --bse bseNifti --t2 t2Nifti --t2mask t2MaskNifti -o dwiEpiPrefix  
        


## ii. Using FSL topup and eddy

If you have two scans in anti-parallel directions (AP and PA), then you can use FSL *eddy_openmp* to correct for both 
Epi and Eddy distortions.

> fsl_topup_epi_eddy -h

    Epi and eddy correction using topup and eddy_openmp/cuda commands in fsl
    For more info, see:
        https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddy/UsersGuide
        https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/topup/TopupUsersGuide
    You can also view the help message:
        `eddy_openmp` or `eddy_cuda`
        `topup`
    
    Usage:
        fsl_topup_epi_eddy [SWITCHES] 
    
    Switches:
        --acqp VALUE:ExistingFile        acuisition parameters file (.txt) containing TWO lines, first
                                         for primary4D (PA), second for secondary4D/3D(AP); required
        --bvals VALUE:str                --bvals primaryBval,secondaryBval --bvals primaryBval if only
                                         one bval file is provided, the second bval file is either
                                         assumed same (secondary4D) or "0.0" (secondary3D); required
        --bvecs VALUE:str                --bvecs primaryBvec,secondaryBvec --bvecs primaryBvec if only
                                         one bvec file is provided, the second bvec file is either
                                         assumed same (secondary4D) or "0.0 0.0 0.0" (secondary3D);
                                         required
        --config VALUE:ExistingFile      config file for FSL eddy tools; see scripts/eddy_config.txt; 
                                         copy this file to your directory, edit relevant sections, 
                                         and provide as --config /path/to/my/eddy_config.txt; required
        --eddy-cuda                      use eddy_cuda instead of eddy_openmp, requires
                                         fsl/bin/eddy_cuda and nvcc in PATH 
        -f VALUE:str                     threshold for fsl bet mask; the default is 0.25
        --imain VALUE:str                --imain primary4D,secondary4D/3D primary: one 4D volume input,
                                         should be PA; secondary: another 3D/4D volume input, should
                                         be AP, which is opposite of primary 4D volume; required
        --mask VALUE:str                 --mask primaryB0mask,secondaryB0mask
        --numb0 VALUE:str                number of b0 images to use from primary and secondary (if
                                         4D): 1 for first b0 only, -1 for all the b0s; the default is
                                         1
        --out VALUE:NonexistentPath      output directory; required
        --whichVol VALUE:str             which volume(s) to correct through eddy: 1(only primary4D) or
                                         1,2(primary4D+secondary4D/3D); the default is 1



a) `--imain primary4D,secondary4D/3D`: You can give two 4D volumes scanned in anti-parallel directions or one 4D and 
    another 3D volume (usually a B0) image.
    

b) `--bvals primaryBval,secondaryBval` or `--bvals primaryBval`: You should provide bvals for your 4D volumes. If you 
    provide only `primaryBval` and other volume is 4D, `secondaryBval` is assumed same. However, if other volume is 3D, 
    `secondaryBval` is assumed `0`.
    
    
c) `--bvecs`: Same rule as of (b) applies for `--bvecs`.
   
    
d) `--mask primaryMask,secondaryMask` or `--mask primaryMask`: You should provide human edited mask for both the volumes. 
    If none of the masks are provided, the workflow creates mask down the way.


e) `--numb0`: FSL `topup` requires b0 images to calculate deformation fields. For all 4D volumes provided, a baseline 
    image is created. You can choose to have one b0 (`--numb0 1`) or all b0 (`--numb0 -1`) to be extracted as baseline
    image from each of the 4D volumes. The baseline images are subsequently provided to `topup`.
    
    
f) `--acqp`: Provide a *.txt* file comprising two lines of acquisition parameters. First line should be for PA (primary4D) 
    and second line should be for AP (secondary4D/3D). If you violate this sequence, the program should still run smoothly. 
    However, the name of the intermediate files might be reversed. The sequence of volumes in the output might be 
    reversed as well.


g) *applytopup*: The basic purpose of this command is to create a mask for *eddy*. If you provide the masks, 
    they are used as input to *applytopup*
    
    
    applytopup --imain=primaryMask --inindex=1 --datatin=my_acq_param.txt --topup=my_topup_results \
    --method=jac --interp=trilinear --out=primaryMaskCorrect
    
    applytopup --imain=secondaryMask --inindex=2 --datatin=my_acq_param.txt --topup=my_topup_results \
    --method=jac --interp=trilinear --out=secondaryMaskCorrect
    
    fslmerge -t topupMask primaryMaskCorrect secondaryMaskCorrect
    fslmaths topupMask -Tmean topupMask
    fslmaths topupMask -bin topupMask

    
If you do not provide the masks, a crude `topupMask` is created from the average of volumes in `topupOut`:

    fslmaths topupOut -Tmean topupOutMean
    bet topupOutMean topupMask -m -n
    
    
h) `--whichVol`: Now that mask for *eddy* is created, you can choose to correct only primary volume (`--whichVol 1`) 
    or both volumes (`--whichVol 1,2`).


    eddy --imain=primary4Dmasked+secondary4D/3Dmasked --mask=topupMask --acqp=acqparams.txt --index=writtenIndex --bvecs=writtenBvecs --bvals=writtenBvals
         --topup=my_topup_results --out=eddy_corrected_data
     
    eddy --imain=primary4Dmasked --mask=topupMask --acqp=acqparams.txt --index=writtenIndex --bvecs=writtenBvecs --bvals=writtenBvals
         --topup=my_topup_results --out=eddy_corrected_data

    
    

**NOTE 1** Any additional arguments to *eddy_openmp*, *topup*, and *applytopup* can be provided by a copy of `scripts/eddy_config.txt` 
file.
    
    
**NOTE 2** All acquisition parameters and indices required for *eddy* binaries are written from provided `--acqp` file 
and based on the assumption that first line is for primaryVolume while second line is for secondaryVolume.


Putting them all together, example usage:

    fsl_topup_epi_eddy --imain primary4D,secondary4D --mask primaryMask,secondaryMask --bvals primaryBval,secondaryBval 
    --bvecs primaryBvec,secondaryBvec --numb0 1 --whichVol 1,2 --acqp acqparams.txt --out /tmp/fsl_epi/ --config /path/to/my/eddy_config.txt
    
    fsl_topup_epi_eddy --imain primary4D,secondary3D --mask primaryMask,secondaryMask 
    --bvals primaryBval --bvecs primaryBvec --numb0 -1 --whichVol 1 --acqp acqparams.txt --out /tmp/fsl_epi/ --config /path/to/my/eddy_config.txt
    
    fsl_topup_epi_eddy --imain primary4D,secondary3D 
    --bvals primaryBval ---bvecs primaryBvec --numb0 -1 --whichVol 1,2 --acqp acqparams.txt --out /tmp/fsl_epi/ --config /path/to/my/eddy_config.txt
    
    fsl_topup_epi_eddy --imain primary4D,secondary3D 
    --bvals primaryBval --bvecs primaryBvec --numb0 1 --whichVol 1 --acqp acqparams.txt --out /tmp/fsl_epi/ --config /path/to/my/eddy_config.txt



# UKFTractography

> ukf -h
    
    ukf.py uses the following default values (if not provided): ['--numTensor', 2, '--stoppingFA', 0.15, '--seedingThreshold', 0.18, 
    '--Qm', 0.001, '--Ql', 70, '--Rs', 0.015, '--stepLength', 0.3, '--recordLength', 1.7, '--stoppingThreshold', 0.1, 
    '--seedsPerVoxel', 10, '--recordTensors']

    ukf.py is a convenient script to run UKFTractography on NIFTI data.
    For NRRD data, you may run UKFTractography executable directly.
    See UKFTractography --help for more default values.
    
    Usage:
        ukf [SWITCHES] 
    
    Meta-switches:
        -h, --help                      Prints this help message and quits
        --help-all                      Prints help messages of all sub-commands and quits
        -v, --version                   Prints the program's version and quits
    
    Switches:
        --bhigh VALUE:str               filter volumes corresponding to bval<=bhigh and run tractography on them only
        --bvals VALUE:ExistingFile      bval file for DWI; required
        --bvecs VALUE:ExistingFile      bvec file for DWI; required
        -i VALUE:ExistingFile           DWI in nifti; required
        -m VALUE:ExistingFile           mask of the DWI in nifti; required
        -o VALUE:str                    output tract file (.vtk); required
        --params VALUE:str              provide comma separated UKF parameters: 
                                        --arg1,val1,--arg2,val2,--arg3,val3 (no spaces)

    

The minimal usage would be:

    ukf -i dwiNifti --bvals bvalFile --bvecs bvecFile -m maskNifti -o /tmp/tracts.vtk
    

However, you can give any additional paramaters with `--params`.

The following parameters are defaults for this script:

ukfdefaults = ['--numTensor', 2, '--stoppingFA', 0.15, '--seedingThreshold', 0.18, '--Qm', 0.001, '--Ql', 70,
'--Rs', 0.015, '--stepLength', 0.3, '--recordLength', 1.7, '--stoppingThreshold', 0.1,
'--seedsPerVoxel', 10, '--recordTensors']

 
You may replace any parameter from the defaults by what you provide with `--params`. For example, if you provide: 

    --params --stepLength,0.4,--stoppingThreshold,0.2,--recordFA,--maxBranchingAngle,45

then parameters passed to UKFTractography will be:

ukfdefaults = ['--numTensor', 2, '--stoppingFA', 0.15, '--seedingThreshold', 0.18, '--Qm', 0.001, '--Ql', 70,
'--Rs', 0.015, **'--stepLength', 0.4,** '--recordLength', 1.7, **'--stoppingThreshold', 0.2,**
'--seedsPerVoxel', 10, '--recordTensors', **'--recordFA'**, **'--maxBranchingAngle','45'**]

Notice the changes in bold.


# FreeSurfer

> nifti_fs -h


    Convenient script to run Freesurfer segmentation
    
    Usage:
        nifti_fs [SWITCHES] 

    Switches:
        --expert VALUE:ExistingFile         expert options to use with recon-all for high-resolution data,
                                            see https://surfer.nmr.mgh.harvard.edu/fswiki/SubmillimeterRecon;
                                            the default is /tmp/pnlNipype/scripts/expert_file.txt
        -f, --force                         if --force is used, any previous output will be overwritten
        -i, --input VALUE:ExistingFile      t1 image in nifti format (nii, nii.gz); required
        -m, --mask VALUE:ExistingFile       mask the t1 before running Freesurfer; if not provided, -skullstrip is
                                            enabled with Freesurfer segmentation
        -n, --nproc VALUE:str               number of processes/threads to use (-1 for all available) for Freesurfer
                                            segmentation; the default is 1
        --nohires                           omit high resolution freesurfer segmentation i.e. do not use -hires flag
        --norandomness                      use the same random seed for certain binaries run under recon-all
        --noskullstrip                      if you do not provide --mask but --input is already masked, 
                                            omit further skull stripping by freesurfer
        -o, --outDir VALUE:str              output directory; required
        --subfields                         FreeSurfer 7 supported -subfields
        --t2 VALUE:ExistingFile             t2 image in nifti format (nii, nii.gz)
        --t2mask VALUE:ExistingFile         mask the t2 before running Freesurfer, if t2 is provided but not its mask,
                                            -skullstrip is enabled with Freesurfer segmentation


Example usage:
    
    nifti_fs -i t1Nifti -m t1Mask -o /tmp/fs/ --subfields
    nifti_fs -i t1Nifti -m t1Mask -o /tmp/fs/ --t2 t2Nifti --t2Mask
    nifti_fs -i t1NiftiAlreadyMasked -o /tmp/fs/ --noskullstrip
    nifti_fs -i t1NiftiAlreadyMasked -o /tmp/fs/ --noskullstrip --expert /my/expert_file.txt
    nifti_fs -i t1NiftiAlreadyMasked -o /tmp/fs/ --noskullstrip --nohires --ncpu 8
    
Note that, `nifti_fs` does not use multiprocessing by default. You can read more about parallel processing with FreeSurfer
segmentation [here](https://surfer.nmr.mgh.harvard.edu/fswiki/ReleaseNotes/#whatsnew).



# FreeSurfer segmentation in T1w space

During FreeSurfer segmentation, `mri/brain.mgz` and `mri/wmparc.mgz` are created in the FreeSurfer `SUBJECTS_DIR`. 
These files are in *.mgz* format. They are in 1mm<sup>3</sup> 256x256x256 space. To obtain the results in original 
anatomical i.e. T1w space:

    cd $SUBJECTS_DIR/<subjid>/mri
    mri_vol2vol --mov brain.mgz --targ rawavg.mgz --regheader --o brain-in-t1w.mgz --no-save-reg
    mri_vol2vol --mov wmparc.mgz --targ rawavg.mgz --regheader --o wmparc-in-t1w.mgz --no-save-reg

You can read more about it [here](https://surfer.nmr.mgh.harvard.edu/fswiki/FsAnat-to-NativeAnat).


# FreeSurfer segmentation in DWI space

During FreeSurfer segmentation, `mri/brain.mgz` and `mri/wmparc.mgz` are created in the FreeSurfer `SUBJECTS_DIR`. 
These files are in *.mgz* format. They are in 1mm<sup>3</sup> 256x256x256 space. 
We would like to view the white matter percellation (*wmparc.mgz*) in DWI space. 

To warp *wmparc.mgz* to DWI space, first we need to compute registration between FreeSurfer space and DWI space. This 
registration can be done in two ways [Direct registration](#-i-direct-registration) and  [Through T2 registration](#-ii-through-t2-registration).
Either way, the FreeSurfer space defining image is *brain.mgz* and the final target is DWI baseline image `bse.nii.gz`.


`FREESURFER_HOME/bin/mri_vol2vol` resamples a volume into another field-of-view using various types of
matrices. Using identity matrix, *brain.mgz* is converted to NIFTI volume `brain.nii.gz`.
 

    brainmgz = self.parent.fsdir / 'mri/brain.mgz'
    wmparcmgz = self.parent.fsdir / 'mri/wmparc.mgz'

    vol2vol('--mov', brainmgz, '--targ', brainmgz, '--regheader', '--o', brain)
    
                        
`FREESURFER_HOME/bin/mri_label2vol` converts a label or a set of labels into a volume. This command is necessary to convert 
*wmparc.mgz* into a NIFTI volume `wmparc.nii.gz`.

    label2vol('--seg', wmparcmgz, '--temp', brainmgz, '--regheader', wmparcmgz, '--o', wmparc)


Now that we have both FreeSurfer space defining image and white matter percellation files in NIFTI format, we can perform
non-linear antsRegistration from *brain.nii.gz* to *bse.nii.gz*. The obtained transforms are applied to *wmparc.nii.gz* 
to get *wmparc.nii.gz* in DWI space.
 

Finally, DWI resolution can be different from FreeSurfer *brain.nii.gz* (i.e T1) resolution. In this case, *wmparc.nii.gz* 
is registered to both DWI and brain resolution. In the `--outDir`, 
look at corresponding files: `wmparcInDwi.nii.gz` `wmparcInBrain.nii.gz`
 
 
The script that does the above is:

> nifti_fs2dwi --help-all

    Registers Freesurfer labelmap to DWI space.

    Usage:
        nifti_fs2dwi [SWITCHES] [SUBCOMMAND [SWITCHES]] 
    
    Switches:
        --bse VALUE:ExistingFile                      masked bse of DWI
        --bvals VALUE:ExistingFile                    bvals file of the DWI; required
        -d, --debug                                   Debug mode, saves intermediate transforms to out/fs2dwi-debug-<pid>
        --dwi VALUE:ExistingFile                      target DWI; required
        --dwimask VALUE:ExistingFile                  DWI mask; required
        -f, --freesurfer VALUE:ExistingDirectory      freesurfer subject directory; required
        --force                                       turn on this flag to overwrite existing output
        -o, --outDir VALUE:str                        output directory; required
    
    Sub-commands:
        direct                                        Direct registration from Freesurfer to B0.; see 'nifti_fs2dwi
                                                      direct --help' for more info
        witht2                                        Registration from Freesurfer to T2 to B0.; see 'nifti_fs2dwi witht2
                                                      --help' for more info
    
    
    ===============================================================
    
    Direct registration from Freesurfer to B0.
    
    Usage:
        nifti_fs2dwi direct  
    
    ================================================================
    
    Registration from Freesurfer to T2 to B0.
    
    Usage:
        nifti_fs2dwi witht2 [SWITCHES] 
    
    Switches:
        --t2 VALUE:ExistingFile          T2 image; required
        --t2mask VALUE:ExistingFile      T2 mask; required


Example usage:

    nifti_fs2dwi --dwi dwiNifti --dwimask dwiMaskNifti -f fsSubDir -o /tmp/fs2dwi/ direct
    nifti_fs2dwi --dwi dwiNifti --dwimask dwiMaskNifti -f fsSubDir -o /tmp/fs2dwi/ witht2 --t2 t2Nifti --t2mask t2MaskNifti
    

## i. Direct registration

Non linear registration:                    *brain.nii.gz* ----->  *b0.nii.gz*

Warp (applying one transform from above):   *wmparc.nii.gz* ----> *wmparcInDwi.nii.gz*

    nifti_fs2dwi --dwi dwiNifti --dwimask dwiMaskNifti -f fsSubDir direct

## ii. Through T2 registration


Rigid registration:                          *brain.nii.gz* ----->  *T2.nii.gz*

Non linear registration:                     *brain.nii.gz* ----->  *b0.nii.gz*

Warp (applying two transforms from above):   *wmparc.nii.gz* ---->  *T2.nii.gz* -----> *wmparcInDwi.nii.gz*
   
    nifti_fs2dwi --dwi dwiNifti --dwimask dwiMaskNifti -f fsSubDir witht2 --t2 t2Nifti --t2mask t2MaskNifti


# White matter query

Queries a whole brain tract file and extracts specific tract files. 

> nifti_wmql -h
 
    Runs tract_querier. Output is <out>/*.vtk

    Usage:
        nifti_wmql [SWITCHES] 

    Switches:
        -f, --fsindwi VALUE:ExistingFile      Freesurfer labelmap in DWI space (nifti); required
        -i, --in VALUE:ExistingFile           tractography file (.vtk or .vtk.gz), must be in RAS space; required
        -n, --nproc VALUE:str                 number of threads to use, if other processes in your computer becomes
                                              sluggish/you run into memory error, reduce --nproc; the default is 4
        -o, --out VALUE:NonexistentPath       output directory; required
        -q, --query VALUE:str                 tract_querier query file (e.g. wmql-2.0.qry); the default is
                                              pnlNipype/scripts/wmql-2.0.qry
    

Example usage:

    nifti_wmql -f `wmparcInDwi.nii.gz` -i tracts.vtk -o /tmp/wmql_output_dir/
    

# Render white matter tracts    

Makes an html page of rendered wmql tracts. Note that execution of this script requires GUI support e.g. NoMachine.

> wmqlqc -h
    
    Usage:
        wmqlqc [SWITCHES] 

    Switches:
        -i VALUE:str                  list of wmql directories, separated by space; list must be
                                      enclosed within quotes; required
        -o VALUE:NonexistentPath      Output directory; required
        -s VALUE:str                  list of subject ids corresponding to wmql directories, separated
                                      by space; list must be enclosed within quotes; required


Example usage:
    
    wmqlqc -i /tmp/wmql_output_dir/ -o /tmp/htmls/ -s caseid
    wmqlqc -i "/tmp/wmql_output_dir1/ /tmp/wmql_output_dir2/" -o /tmp/htmls -s "caseid1 caseid2"



# Extract measures from tracts

Extracts diffusion measures from white matter query tracts.

> /path/to/Slicer --launch FiberTractMeasurements --help

    /SlicerDMRI/lib/Slicer-4.11/cli-modules/./FiberTractMeasurements
     [--returnparameterfile <std::string>]
     [--processinformationaddress <std::string>]
     [--xml]
     [--echo]
     [--moreStatistics]
     [--printAllStatistics]
     [-s <Comma|Space|Tab>]
     [-f <Row_Hierarchy|Column_Hierarchy>]
     [--outputfile <std::string>]
     [--inputdirectory <std::string>]
     [--fiberHierarchy <std::string>] ...
     [-i <Fibers_Hierarchy|Fibers_File_Folder>]
     [--]
     [--version]
     [-h]


Example usage:

    /path/to/Slicer --launch FiberTractMeasurements \
    --inputtype Fibers_File_Folder \
    --format Column_Hierarchy \
    --separator Comma \
    --inputdirectory /tmp/wmql_output_dir/ \
    --outputfile  /tmp/tractMeasures.csv


**NOTE** The above `Slicer` is representative of one that has knowledge of all your extension modules. If it is a shared 
`Slicer` not installed in your home directory, then making it knowledgeable of all your extension modules can be a little 
tricky. You may need something akin to:

    /path/to/Slicer --launcher-additional-settings /path/to/Slicer-12345.ini --launch FiberTractMeasurements


# 800 cluster white matter percellation

The task is performed by `wm_apply_ORG_atlas_to_subject.sh` script from whitematteranalysis package. 
Details about it can be found [here](https://github.com/pnlbwh/luigi-pnlpipe/blob/769f97184899cc03908ed403ac59051f2cded248/docs/TUTORIAL.md#wma800). 
Installation instruction can be found [here](https://github.com/pnlbwh/luigi-pnlpipe/blob/769f97184899cc03908ed403ac59051f2cded248/docs/README.md#whitematteranalysis).

Example usage:

    wm_apply_ORG_atlas_to_subject.sh -i tract.vtk -a /path/to/ORG-Atlases-1.2 \
    -s /path/to/Slicer -m "/path/to/Slicer --launch FiberTractMeasurements" -n 4 \
    -o /tmp/wma800/ -x 1 -d 1


# Additional scripts

New addition to pnlNipype is a workflow for finding various quality attributes of DWMRI. It can be found in 
`scripts/DWIqc/dwi_quality.py` and `scripts/DWIqc/dwi_quality_batch.py`.

> scritps/DWIqc/dwi_quality.py -h

    This script finds various DWMRI quality attributes:
    
     1. min_i{(b0<Gi)/b0} image and its mask
     2. negative eigenvalue mask
     3. fractional anisotropy FA, mean diffusivity MD, and mean kurtosis MK
     4. masks out of range FA, MD, and MK
     5. prints histogram for specified ranges
     6. gives a summary of analysis
     7. performs ROI based analysis:
        labelMap is defined in template space
        find b0 from the dwi
        register t2 MNI to dwi through b0
        find N_gradient number of stats for each label
        make a DataFrame with labels as rows and stats as columns
    
    Looking at the attributes, user should be able to infer quality of the DWMRI and
    changes inflicted upon it by some process.
    
    Usage:
        dwi_quality.py [SWITCHES]
    
    Meta-switches:
        -h, --help                                Prints this help message and quits
        --help-all                                Prints help messages of all sub-commands and quits
        -v, --version                             Prints the program's version and quits
    
    Switches:
        --bval VALUE:ExistingFile                 bval for nifti image
        --bvec VALUE:ExistingFile                 bvec for nifti image
        --fa VALUE:str                            [low,high] (include brackets, no space), fractional anisotropy values outside the range are
                                                  masked; the default is [0,1]
        -i, --input VALUE:ExistingFile            input nifti/nrrd dwi file; required
        -l, --labelMap VALUE:ExistingFile         labelMap in standard space
        -m, --mask VALUE:ExistingFile             input nifti/nrrd dwi file; required
        --md VALUE:str                            [low,high], (include brackets, no space), mean diffusivity values outside the range are
                                                  masked; the default is [0,0.0003]
        --mk VALUE:str                            [low,high] (include brackets, no space), mean kurtosis values outside the range are masked,
                                                  requires at least three shells for dipy model; the default is [0,0.3]
        -n, --name VALUE:str                      labelMap name
        -o, --outDir VALUE:ExistingDirectory      output directory; the default is input directory
        -t, --template VALUE:ExistingFile         t2 image in standard space (ex: T2_MNI.nii.gz)


Any feedback on this new workflow is welcome.
