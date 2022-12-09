# 3DiGAN

Code for **3D** aware **i**mplicit **G**enerative **A**dversial **N**etwork.

This repository extends a [lightweight generative network](https://github.com/lucidrains/lightweight-gan) to learn a distribution of 2D image UV textures wrapped on an underlying geometry. Given a mesh prior, the generator synthesises UV appearance textures which are then rendered on top of the geometry. Colored points are sampled from the mesh and displaced along the mesh normal according to the last UV texture channel, which operates as a displacement map.

As stated above, this code builds on top of an implementation by GitHub user [lucidrains](https://github.com/lucidrains). The mentioned code license is provided in the below toggle.

<details>
<summary> <b>Lightweight GAN</b> license </summary>

```markdown
MIT License

Copyright (c) 2021 Phil Wang

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
</details>


## Installations

Clone this repository and install the dependencies with the below commands.
```bash
git clone https://github.com/maximeraafat/3DiGAN.git
pip install -r 3DiGAN/requirements.txt
```

The point-based rendering framework utilises [PyTorch3D](https://pytorch3d.org). Checkout the steps described in their provided [installation instruction set](https://github.com/facebookresearch/pytorch3d/blob/main/INSTALL.md) with matching versions of **PyTorch**, and **CUDA** if applicable.

Learning a human appearance model requires an underlying geometry prior : **3DiGAN** leverages the SMPLX parametric body model. Download the body model `SMPLX_NEUTRAL.npz` and corresponding UVs `smplx_uv.obj` from the [SMPLX project page](https://smpl-x.is.tue.mpg.de) into a shared folder. Instead of storing SMPLX meshes individually, we store SMPLX parameters into a **npz** file. The instructins on how to capture SMPLX parameters for a dataset of single-view full body human images using [PIXIE](https://github.com/YadiraF/PIXIE) will be made available.


## Training

Our code's purpose is the learning and synthesis of novel appearances; here we provide instructions for two different scenarios.

### Human Appearance

Given a large dataset of full body humans (see [SHHQ](https://github.com/stylegan-human/StyleGAN-Human/blob/main/docs/Dataset.md)) and corresponding SMPLX parameters, execute the following command.

```bash
python 3DiGAN/main.py --data <path/to/dataset> \
                      --models_dir <path/to/output/models> \
                      --results_dir <path/to/output/results> \
                      --name <run/name> \
                      --render \
                      --smplx_model_path <path/to/smplx>
```

The `--smplx_model_path` option provides the path to the SMPLX models folder, and requires an **npz** file containing all the estimated SMPLX parameters for each image in the dataset. See the [installations](#installations) section for details. The **npz** file must be accessible either by

1. renaming the **npz** file to `dataset.npz` and including it into the dataset folder under `<path/to/dataset>`, or by
2. providing the path to the **npz** file with `--labelpath <path/to/npz>`

### Arbitrary Geometry Appearance

To synthesise appearance for an arbitrary fixed geometry prior, provide the path to a mesh **obj** file containing UVs with the `--mesh_obj_path` option.

```bash
python 3DiGAN/main.py --data <path/to/dataset> \
                      --models_dir <path/to/output/models> \
                      --results_dir <path/to/output/results> \
                      --name <run/name> \
                      --render \
                      --mesh_obj_path <path/to/obj>
```

The `--mesh_obj_path` option requires a **json** file contaning estimated or ground truth camera azimuth and elevations for each image in the dataset. Note that the focal length to our point rendering camera is fixed to 10. Analogously to the human apperance modelling section, the **json** file must be accessible either by

1. renaming the **json** file to `dataset.json` and including it into the dataset folder under `<path/to/dataset>`, or by
2. providing the path to the **json** file with `--labelpath <path/to/json>`

An example dataset of renders of the PyTorch3D cow mesh with corresponding **json** file containing camera poses labels is accessible [here](https://drive.google.com/file/d/1RxPwHNNr-7jyG7gU4ulzPAyNR1vcKAnD/view?usp=share_link), and the cow **obj** file is accessible [under this link](https://dl.fbaipublicfiles.com/pytorch3d/data/cow_mesh/cow.obj).


## Generation

To synthesise new human appearances from a trained generator, execute this command.

```bash
python 3DiGAN/main.py --generate \
                      --models_dir <path/to/output/models> \
                      --results_dir <path/to/output/results> \
                      --name <run/name> \
                      --render \
                      --labelpath <path/to/npz> \
                      --smplx_model_path <path/to/smplx>
```

Unlike for training, generation requires the `--labelpath` option since the dataset is not provided. To synthesise arbitrary geometry appearances, replace the `--smplx_model_path` option for `--mesh_obj_path` and adapt `--labelpath`.

## Settings

This section discusses the relevant command line arguments. The code follows a similar structure to the original [lightweight GAN](https://github.com/lucidrains/lightweight-gan) implementation and supports the same options, while adding arguments for the rendering environment. Please visit the parent repository for further details.

* `--render_size` : square rendering resolution, by default set to `256`. This flag does not replace `--image_size` (also by default set to `256`), which is the generated square UV map resolution.
* `--render` : whether to render the learned generated output. Without this flag, the code is essentially a copy of lightweight GAN.
* `--renderer` : set by default to `default`, defines which point renderer to use. Has to be one of of `default or `pulsar`.
* `--nodisplace` : call this flag to learn RGB appearances without a fourth displacement channel.
* `--num_points` : number of points sampled from the underlying mesh geometry, by default set to `10**5`.
* `--gamma` : point transparency coefficient for pulsar (defined between 1e-5 and 1), by default set to `1e-3`.
* `--radius` : point radius, set by default to `0.01` for the default renderer and to `0.0005` for pulsar.
* `--smplx_model_path` : path to the SMPLX models folder.
* `--mesh_obj_path` : path to the underlying **obj** mesh file.
* `--labelpath` : path to the **npz**, respectively **json** json file containing smplx parameters or camera poses necessary for rendering.

Note that the generated UV textures are currently concatenated into 4 channel RGBD images rather than RGB images, plus a separate displacement texture map. The `--displacement` and `--greyscale` options are not support when calling the `--render` flag.

The `--show_progress` and `--generate_interpolation` flags from the original parent implementation are supported, but operate in the UV space rather than in the render space.


## Upcoming
- [ ] Instructions for extracting SMPLX parameters with PIXIE on SHHQ.
- [ ] Release **npz** file containing SMPLX parameters for SHHQ subjects, estimated with PIXIE.
