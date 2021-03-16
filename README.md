# NYU-Singularity

Singularity is a docker type container which acts as mini ubuntu with all its file system, but it contributes only one file. NYU limits the memory we get (for example /home/pp1953/ has gone down from 20Gb to 2Gb), and hence installing a conda environment is not possible on /home/pp1953/. Another problem is the number of files large filesystems can hold, e.g., 10Gb /scratch/pp1953 can not keep a 5Gb folder with 10,000+ files. Hence we use singularity, a 5Gb single file for anaconda, and since it's mini ubuntu, it can load other folders. 
For me it was a dataset having size of 5.6GB with 100s of files which I couldnt extract to `scratch/pp1953` so the following is the only solution 

## Setting up the Environment 
Note everything we will be doing is applicable only in your working directory, in my case its `/scratch/pp1953/temp/`
You fully fleged anaconda environemnt is around ~2-3Gb and contains 100+ files. We shall be using a 5GB mini ubuntu (a replacement of your /home/pp1953/ which has only 2GB memeory) and can easily install conda environement inside it. 

### STEP 0 : Check if singularity exists 
```
which singularity 
singularity version
```
### STEP 1 : setting up container (mini ubuntu)
Do all of these in your scratch file system
```
cd /scratch/pp1953/temp/
cp -rp /scratch/work/public/overlay-fs-ext3/overlay-5GB-200K.ext3.gz .
gunzip overlay-5GB-200K.ext3.gz
```
This file `overlay-5GB-200K.ext3` will be your mini ubuntu 1 file. Everything will occur inside it 


### STEP 2 : setting up Conda environement inside container
Open the container in bash mode (interactive shell of the mini ubuntu)
```
singularity exec --overlay overlay-5GB-200K.ext3 /scratch/work/public/singularity/cuda11.0-cudnn8-devel-ubuntu18.04.sif /bin/bash
```
^^ `/scratch/work/public/singularity/cuda11.0-cudnn8-devel-ubuntu18.04.sif` is the cuda libraries for pytorch. NYU recoomends `/scratch/work/public/singularity/cuda10.1-cudnn7-devel-ubuntu18.04.sif` for tensorflow installation


Installing conda environment (or follow the steps for anaconda step here : https://github.com/ppriyank/Prince-Set-UP)
``` 
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
sh Miniconda3-latest-Linux-x86_64.sh -b -p /ext3/miniconda3
export PATH=/ext3/miniconda3/bin:$PATH
conda update -n base conda -y
```
create a wrapper script /ext3/env.sh  or env.sh (which is like ~/.bash_profile for you system )
```
#!/bin/bash

source /ext3/miniconda3/etc/profile.d/conda.sh
export PATH=/ext3/miniconda3/bin:$PATH
```
**Exit the container** and relaunch it again ~ loading a new terminal where conda environement is available 

### STEP 3 : Installing files 
Launch the environemnt
```
singularity exec --overlay overlay-5GB-200K.ext3 /scratch/work/public/singularity/cuda10.1-cudnn7-devel-ubuntu18.04.sif /bin/bash
```
load the .bash_profile : `source /ext3/env.sh` or `source env.sh`

Sanity check if conda is installed properly : all should return something 
```
which python
## /ext3/miniconda3/bin/python
which pip   
## /ext3/miniconda3/bin/pip
python --version
## Python 3.8.5
which conda
## /ext3/miniconda3/bin/conda
```
Install pytorch : follow this or steps here : https://github.com/ppriyank/Prince-Set-UP

```
pip install torch==1.7.0+cu110 torchvision==0.8.1+cu110 torchaudio===0.7.0 -f https://download.pytorch.org/whl/torch_stable.html
pip install jupyter jupyterhub pandas matplotlib scipy
pip install scikit-learn scikit-image Pillow
conda clean --all --yes
```
`conda clean --all --yes` basically cleans the download tars, unnecessary cache files when we only 5 GB in our file system. 

Used up memory : 
`du -sh /ext3/`

**Exit the container** and rename it, and your portable mini ubuntu is read :D 
```mv overlay-5GB-200K.ext3 chikki_chikkah_pytorch1.7.0-cuda11.0.ext3```

### STEP 4 : Sanity check for available GPUs

Request a GPU using the moethodology : https://github.com/ppriyank/Prince-Set-UP
Verify GPUs : change the name `chikki_chikkah_pytorch1.7.0-cuda11.0.ext3` and path of the environement file `source env.sh` according to your set up 
```
singularity exec --nv \
	    --overlay /scratch/pp1953/trial/chikki_chikkah_pytorch1.7.0-cuda11.0.ext3:ro \
	    /scratch/work/public/singularity/cuda11.0-cudnn8-devel-ubuntu18.04.sif \
	    bash -c 'source env.sh; python -c "import torch; print(torch.__file__); print(torch.__version__); print(torch.cuda.device_count())"'
```


### STEP 5 : creating a mount for your mini ubuntu (your dataset or large folder)

Take the zip that you can extract to scratch (on the login node): 
```
mkdir -p  /state/partition1/pp1953
cd /state/partition1/pp1953/
unzip  /archive/pp1953/mars_test.zip
find bbox_test -type d -exec chmod 755 {} \;
find bbox_test -type f -exec chmod 644 {} \;

mksquashfs bbox_test bbox_test.sqf -keep-as-directory
mv bbox_test.sqf /scratch/pp1953/
```

This will create a single file `bbox_test.sqf` which can be read by your singularity file (mini ubuntu) and its only file :D
```
singularity exec --nv \
	    --overlay /scratch/gg2501/trial/pytorch1.7.0-cuda11.0.ext3:ro \
	    --overlay  /scratch/pp1953/bbox_test.sqf:ro \
	    /scratch/work/public/singularity/cuda11.0-cudnn8-devel-ubuntu18.04.sif \
	     /bin/bash
```

This is a GPU loaded mini ubuntu and your "huge folder" mounted on root "/" 
```
ls /bbox_test
```
^^ this `/bbox_test` can not be extracted on scratch but is mounted on this mini ubuntu :) 

For spark submit jobs
replace `/bin/bash` with 
`bash -c 'source env.sh; python code.py'`

ENJOY!!






