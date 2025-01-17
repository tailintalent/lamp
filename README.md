# LAMP: Learning Controllable Adaptive Simulation for Multi-resolution Physics (ICLR 2023 Notable-Top-25%)

[Paper](https://openreview.net/forum?id=PbfgkZ2HdbE) | [arXiv](https://arxiv.org/abs/2305.01122) | [Poster](https://github.com/snap-stanford/lamp/blob/master/assets/lamp_poster.pdf) | [Slides](https://docs.google.com/presentation/d/1cMRGe2qNIrzSNRTUtbsVUod_PvyhDHcHzEa8wfxiQsw/edit?usp=sharing) | [Tweet](https://twitter.com/tailin_wu/status/1653253117671272448) | [Project Page](https://snap.stanford.edu/lamp/) 

Official repo for the paper [Learning Controllable Adaptive Simulation for Multi-resolution Physics](https://openreview.net/forum?id=PbfgkZ2HdbE)<br />
[Tailin Wu*](https://tailin.org/), [Takashi Maruyama*](https://sites.google.com/view/tmaruyama/home), [Qingqing Zhao*](https://cyanzhao42.github.io/), [Gordon Wetzstein](https://stanford.edu/~gordonwz/), [Jure Leskovec](https://cs.stanford.edu/people/jure/)<br />
ICLR 2023 **Notable-Top-25%**. 

It is the first fully DL-based surrogate model that jointly learns the evolution model, and optimizes spatial resolutions to reduce computational cost, learned via reinforcement learning. 

We demonstrate that LAMP is able to adaptively trade-off computation to improve long-term prediction error, by performing spatial refinement and coarsening of the mesh. LAMP outperforms state-of-the-art (SOTA) deep learning surrogate models, with an average of 33.7% error reduction for 1D nonlinear PDEs, and outperforms SOTA MeshGraphNets + Adaptive Mesh Refinement in 2D mesh-based simulations.

<a href="url"><img src="https://github.com/snap-stanford/lamp/blob/master/assets/lamp_architecture.png" align="center" width="700" ></a>

Learned remeshing & evolution by LAMP:
<a href="url"><img src="https://github.com/snap-stanford/lamp/blob/master/assets/gif-lamp.gif" align="center" width="300" ></a>

## Installation

1. First clone the directory. Then run the following command to initialize the submodules:

```code
git submodule init; git submodule update
```

(If showing error of no permission, need to first [add a new SSH key to your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account).)

2. Install dependencies.

First, create a new environment using [conda](https://docs.conda.io/en/latest/miniconda.html) (with python >= 3.7). Then install pytorch, torch-geometric and other dependencies as follows (the repository is run with the following dependencies. Other version of torch-geometric or deepsnap may work but there is no guarentee.)

Install pytorch (replace "cu113" with appropriate cuda version. For example, cuda11.1 will use "cu111"):
```code
pip install torch==1.10.2+cu113 torchvision==0.11.3+cu113 torchaudio==0.10.2+cu113 -f https://download.pytorch.org/whl/torch_stable.html
```

Install torch-geometric. Run the following command:
```code
pip install torch-scatter==2.0.9 -f https://data.pyg.org/whl/torch-1.10.2+cu113.html
pip install torch-sparse==0.6.12 -f https://data.pyg.org/whl/torch-1.10.2+cu113.html
pip install torch-geometric==1.7.2
pip install torch-cluster==1.5.9 -f https://data.pyg.org/whl/torch-1.10.2+cu113.html
```

Install other dependencies:
```code
pip install -r requirements.txt
```

If wanting to use wandb (--wandb=True), need to set up wandb, following [this link](https://docs.wandb.ai/quickstart).

If wanting to run 2d mesh-based simulation, FEniCS needs to be installed:

```code
conda install -c conda-forge fenics
```


## Dataset

The dataset files can be downloaded via [this link](https://drive.google.com/drive/folders/1ld5I86mPC7wWTxPhbCtG2AcH0vLW3o25?usp=share_link). 
* To run 1D experiment, download the files under "mppde1d_data/" in the link into the "data/mppde1d_data/" folder in the local repo. 
* To run 2D mesh-based experiment, download the files under "arcsimmesh_data/" in the link into the "data/arcsimmesh_data/" folder in the local repo. Script for data generation is also provided ("datasets/datagen_square.ipynb".) To run the script, compile [ARCSim v0.2.1](http://graphics.berkeley.edu/resources/ARCSim/#:~:text=ArcSim%20is%20a%20simulation,detail%20of%20the%20simulated%20objects) and place the script in ARCSim folder. The detailed explanation for the attributes for the mesh-based dataset are provided under [datasets/README.md](https://github.com/snap-stanford/lamp/tree/master/datasets).


## Training

Below we provide example commands for training LAMP.

### 1D nonlinear PDE:

First, **pre-train** the evolution model for 1D:

```code
python train.py --exp_id=evo-1d --date_time=2023-01-01 --dataset=mppde1df-E2-100-nt-250-nx-200 --time_interval=1 --data_dropout=node:0-0.3:0.1 --latent_size=64 --n_train=-1 --save_interval=5 --test_interval=5 --algo=gnnremesher --rl_coefs=None --input_steps=1 --act_name=silu --multi_step=1^2:0.1^3:0.1^4:0.1 --temporal_bundle_steps=25 --use_grads=False --is_y_diff=False --loss_type=mse --batch_size=16 --val_batch_size=16 --epochs=50 --opt=adam --weight_decay=0 --seed=0 --id=0 --verbose=1 --n_workers=0 --gpuid=0
```

The learned model will be saved under `./results/{--exp_id}_{--date_time}/`, where the `{--exp_id}` and `{--date_time}` are specified in the above command. The filename has the format of `*{hash}_{machine_name}.p`, e.g. "mppde1df-E2-100-nt-250-nx-200_train_-1_algo_gnnremesher_..._Hash_mhkVkAaz_ampere3.p", then the `{hash}` is `mhkVkAaz` and `{machine_name}` is `ampere3`, where the `{hash}` is uniquely determined by **all** the argument settings in the [argparser.py](https://github.com/snap-stanford/lamp/blob/master/argparser.py) (therefore, as long as any argument setting is different, the filename will be different and will not overwrite each other).

Then, **jointly train** the remeshing model via reinforcement learning (RL) and the evolution model. The `--load_dirname` below should use folder name `{exp_id}_{date_time}` where the evolution model is located (as specified above), and the `--load_filename` should use part of the filename that can uniquely identify this model file, and should include the `{hash}` of this model.

```code
python train.py --load_dirname=evo-1d_2023-01-01 --load_filename=Q66bz42y --exp_id=rl-1d --date_time=2023-01-02 --wandb_project_name=rl-1d_2023-01-02 --wandb=True --dataset=mppde1df-E2-100-nt-250-nx-200 --time_interval=1 --data_dropout=None --latent_size=64 --n_train=-1 --input_steps=1 --act_name=elu --multi_step=1^2:0.1^3:0.1^4:0.1 --temporal_bundle_steps=25 --use_grads=False --is_y_diff=False --loss_type=mse --batch_size=128 --val_batch_size=128 --epochs=30 --opt=adam --weight_decay=0 --seed=0 --verbose=1 --n_workers=0 --gpuid=7 --algo=srlgnnremesher --reward_mode=lossdiff+statediff --reward_beta=0-0.5 --rl_data_dropout=uniform:2 --min_edge_size=0.0014 --rl_horizon=4 --reward_loss_coef=5 --rl_eta=1e-2 --actor_lr=5e-4 --value_lr=1e-4 --value_num_pool=1 --value_pooling_type=global_mean_pool --value_latent_size=32 --value_batch_norm=False --actor_batch_norm=True --rescale=10 --edge_attr=True --rl_gamma=0.9 --value_loss_coef=0.5 --max_grad_norm=2 --is_single_action=False --value_target_mode=vanilla --wandb_step_plot=100 --wandb_step=20 --save_iteration=1000 --save_interval=1 --test_interval=1 --gpuid=3 --lr=1e-4 --actor_critic_step=200 --evolution_steps=200 --reward_condition=True --max_action=20 --rl_is_finetune_evolution=True --rl_finetune_evalution_mode=policy:fine  --id=0
```

### 2D mesh-based simulation:

**Pre-train** the evolution model for 2D (need to have FEniCS installed, see "Installation" section:

```code
export OMP_NUM_THREADS=6; python train.py --exp_id=evo-2d  --date_time=2023-01-01 --dataset=arcsimmesh_square_annotated --time_interval=2 --data_dropout=None --n_train=-1 --save_interval=5 --algo=gnnremesher-evolution --rl_coefs=None --input_steps=2 --act_name=silu --multi_step=1 --temporal_bundle_steps=1 --edge_attr=True --use_grads=False --is_y_diff=False --loss_type=l2 --batch_size=10 --val_batch_size=10 --latent_size=56 --n_layers=8 --noise_amp=1e-2 --correction_rate=0.9 --epochs=100 --opt=adam --weight_decay=0  --is_mesh=True --seed=0 --id=0 --verbose=2 --test_interval=2 --n_workers=20 --gpuid=0
```

Then, **jointly train** the remeshing model via RL and the evolution model:

```code
export OMP_NUM_THREADS=6; python train.py --exp_id=2d_rl --wandb_project_name=2d_rerun --wandb=True --date_time=2023-02-26 --dataset=arcsimmesh_square_annotated_coarse_minlen008_interp_500 --time_interval=2 --n_train=-1 --latent_size=64 --load_dirname=evo-2d_2023_02_18 --load_filename=9UQLIKKc_ampere1 --input_steps=2 --act_name=elu --temporal_bundle_steps=1 --use_grads=False --is_y_diff=True --loss_type=l2 --epochs=300 --opt=adam --weight_decay=0 --verbose=1 --algo=srlgnnremesher --reward_mode=lossdiff+statediff --rl_data_dropout=None --min_edge_size=0.04 --actor_lr=5e-4 --value_lr=1e-4 --value_num_pool=1 --value_pooling_type=global_mean_pool --value_latent_size=64 --value_batch_norm=False --actor_batch_norm=True --rescale=10 --edge_attr=True --rl_gamma=0.9 --value_loss_coef=0.5 --max_grad_norm=20 --is_single_action=False --value_target_mode=vanilla --wandb_step_plot=50 --wandb_step=2 --id=0 --save_iteration=500 --save_interval=1 --test_interval=1 --is_mesh=True --is_unittest=False --rl_horizon=6 --multi_step=6 --rl_eta=2e-2 --reward_beta=0 --reward_condition=True --max_action=20 --rl_is_finetune_evolution=True --lr=1e-4 --actor_critic_step=200 --evolution_steps=100 --rl_finetune_evalution_mode=policy:fine --wandb=True --batch_size=64 --val_batch_size=64 --n_workers=6 --reward_loss_coef=1000 --evl_stop_gradient=True --noise_amp=0.01 --gpuid=5 --is_eval_sample=True --seed=256 --n_train=:-1 --soft_update=False --fine_tune_gt_input=True --policy_input_feature=coords --skip_coarse=False  --skip_flip=True  --processor_aggr=mean  --fix_alt_evolution_model=True
```

For commands for **baseline** models in 1D, see the README in [./MP_Neural_PDE_Solvers/](https://github.com/tailintalent/MP_Neural_PDE_Solvers).

We also provide pre-trained evolution models directly for RL training [here](https://drive.google.com/drive/folders/1ioR5gjYeQaNQMvqrdMYMZI-0n5k5UM8f). Put the folders in the Google doc (e.g., "evo-1d_2023-01-01" for pre-trained evolution model for 1d, "evo-2d_2023-02-18" for pre-trained evolution model for 2d) under the ./results/ folder, and can then use the RL commands above to perform joint training.

## Analysis

* For 1D experiments, to analyze the pretrained evolution model for LAMP, use [analysis_1D_evo.ipynb](https://github.com/snap-stanford/lamp/blob/master/analysis_1d_evo.ipynb).

* For 1D experiments, to analyze the full model for LAMP and the baselines, use [analysis_1D_full.py](https://github.com/snap-stanford/lamp/blob/master/analysis_1d_full.py).

* For 1D experiments, to analyze the baseline models (MP-PDE, FNO, CNN), use [./MP_Neural_PDE_Solvers/analysis.ipynb](https://github.com/tailintalent/MP_Neural_PDE_Solvers/blob/master/analysis.ipynb).

* For 2D experiments, to analyze the pretrained evolution model for LAMP, use [analysis_2D_evo.ipynb](https://github.com/snap-stanford/lamp/blob/master/analysis_2d_evo.ipynb).

* For 2D experiments, to analyze the full model for LAMP and the baselines, use [analysis_2D_full.py](https://github.com/snap-stanford/lamp/blob/master/analysis_2d_full.py) and [analysis_2d_rl.ipynb](https://github.com/snap-stanford/lamp/blob/master/analysis_2d_rl.ipynb).

## Visualization:

Example visualization of learned remeshing & evolution:

LAMP:

<a href="url"><img src="https://github.com/snap-stanford/lamp/blob/master/assets/gif-lamp.gif" align="center" width="300" ></a>

LAMP (no remeshing):

<a href="url"><img src="https://github.com/snap-stanford/lamp/blob/master/assets/gif-lamp_no_remeshing.gif" align="center" width="300" ></a>

MeshGraphNets + ground-truth remeshing:

<a href="url"><img src="https://github.com/snap-stanford/lamp/blob/master/assets/gif-MeshGraphNets+gt remeshing.gif" align="center" width="300" ></a>


MeshGraphNets + heuristic remeshing:

<a href="url"><img src="https://github.com/snap-stanford/lamp/blob/master/assets/gif-MeshGraphNets+heuristic remeshing.gif" align="center" width="300" ></a>


Ground-truth (fine-grained):

<a href="url"><img src="https://github.com/snap-stanford/lamp/blob/master/assets/gif-ground_truth.gif" align="center" width="300" ></a>

## Related Projects

* [LE-PDE](https://github.com/snap-stanford/le_pde) (NeurIPS 2022): Accelerate the simulation and inverse optimization of PDEs. Compared to state-of-the-art deep learning-based surrogate models (e.g., FNO, MP-PDE), it is up to 15x improvement in speed, while achieving competitive accuracy.

## Citation
If you find our work and/or our code useful, please cite us via:

```bibtex
@inproceedings{wu2023learning,
  title={Learning Controllable Adaptive Simulation for Multi-resolution Physics},
  author={Tailin Wu and Takashi Maruyama and Qingqing Zhao and Gordon Wetzstein and Jure Leskovec},
  booktitle={The Eleventh International Conference on Learning Representations},
  year={2023},
  url={https://openreview.net/forum?id=PbfgkZ2HdbE}
}
```
