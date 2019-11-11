# geotensor

[![MIT License](https://img.shields.io/badge/license-MIT-green.svg)](https://opensource.org/licenses/MIT)
![Python 3.7](https://img.shields.io/badge/Python-3.7-blue.svg)
[![repo size](https://img.shields.io/github/repo-size/xinychen/geotensor.svg)](https://github.com/xinychen/geotensor/archive/master.zip)
[![GitHub stars](https://img.shields.io/github/stars/xinychen/geotensor.svg?logo=github&label=Stars&logoColor=white)](https://github.com/xinychen/geotensor)


## Motivation

- **Color image inpainting**: Various missing patterns.

<table>
  <tr>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_mar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_mar.jpg?size=150" width="150px;" alt="Missing at random (MAR)"/><br /><sub><b>Missing at random (MAR)</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_rmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_rmar.jpg?size=150" width="150px;" alt="Row-wise MAR"/><br /><sub><b>Row-wise MAR</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_cmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_cmar.jpg?size=150" width="150px;" alt="Column-wise MAR"/><br /><sub><b>Column-wise MAR</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_rcmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_rcmar.jpg?size=150" width="150px;" alt="(Row, column)-wise MAR"/><br /><sub><b>(Row, column)-wise MAR</b></sub></a><br /></td>
  </tr>
</table>

- **Low-rank tensor completion**: Characterizing images with graph (e.g., adjacent smoothness matrix based graph regularizer).

## Implementation

- Proposed Models

  - [GLTC-NN (Nuclear Norm)](https://nbviewer.jupyter.org/github/xinychen/geotensor/blob/master/GLTC-NN.ipynb)
  - [GLTC-Geman (nonconvex)](https://nbviewer.jupyter.org/github/xinychen/geotensor/blob/master/GLTC-Geman.ipynb)
  - [GLTC-Laplace (nonconvex)](https://nbviewer.jupyter.org/github/xinychen/geotensor/blob/master/GLTC-Laplace.ipynb)
  - [GTC (without low-rank assumption)](https://nbviewer.jupyter.org/github/xinychen/geotensor/blob/master/GTC.ipynb)


One notable thing is that unlike the complex equations in our models, our Python implementation (relies on `numpy`) is extremely easy to work with. Take **`GLTC-Geman`** as an example, its kernel only has few lines:

```python
def supergradient(s_hat, lambda0, theta):
    """Supergradient of the Geman function."""
    return (lambda0 * theta / (s_hat + theta) ** 2)

def GLTC_Geman(dense_tensor, sparse_tensor, alpha, beta, rho, theta, maxiter):
    """Main function of the GLTC-Geman."""
    dim0 = sparse_tensor.ndim
    dim1, dim2, dim3 = sparse_tensor.shape
    dim = np.array([dim1, dim2, dim3])
    binary_tensor = np.zeros((dim1, dim2, dim3))
    binary_tensor[np.where(sparse_tensor != 0)] = 1
    tensor_hat = sparse_tensor.copy()
    
    X = np.zeros((dim1, dim2, dim3, dim0)) # \boldsymbol{\mathcal{X}} (n1*n2*3*d)
    Z = np.zeros((dim1, dim2, dim3, dim0)) # \boldsymbol{\mathcal{Z}} (n1*n2*3*d)
    T = np.zeros((dim1, dim2, dim3, dim0)) # \boldsymbol{\mathcal{T}} (n1*n2*3*d)
    for k in range(dim0):
        X[:, :, :, k] = tensor_hat
    
    D1 = np.zeros((dim1 - 1, dim1)) # (n1-1)-by-n1 adjacent smoothness matrix
    for i in range(dim1 - 1):
        D1[i, i] = -1
        D1[i, i + 1] = 1
    D2 = np.zeros((dim2 - 1, dim2)) # (n2-1)-by-n2 adjacent smoothness matrix
    for i in range(dim2 - 1):
        D2[i, i] = -1
        D2[i, i + 1] = 1
    
    for iters in range(maxiter):
        for k in range(dim0):
            u, s, v = np.linalg.svd(ten2mat(X[:, :, :, k] + T[:, :, :, k] / rho, k), full_matrices = 0)
            for i in range(len(np.where(s > 0)[0])):
                s[i] = max(s[i] - supergradient(s[i], alpha / rho, theta) / rho, 0)
            Z[:, :, :, k] = mat2ten(np.matmul(np.matmul(u, np.diag(s)), v), dim, k)
            var = ten2mat(rho * Z[:, :, :, k] - T[:, :, :, k], k)
            if k == 0:
                var0 = mat2ten(np.matmul(inv(beta * np.matmul(D1.T, D1) + rho * np.eye(dim1)), var), dim, k)
            elif k == 1:
                var0 = mat2ten(np.matmul(inv(beta * np.matmul(D2.T, D2) + rho * np.eye(dim2)), var), dim, k)
            else:
                var0 = Z[:, :, :, k] - T[:, :, :, k] / rho
            X[:, :, :, k] = np.multiply(1 - binary_tensor, var0) + np.multiply(binary_tensor, sparse_tensor)
        tensor_hat = np.mean(X, axis = 3)
        for k in range(dim0):
            T[:, :, :, k] = T[:, :, :, k] + rho * (X[:, :, :, k] - Z[:, :, :, k])
            X[:, :, :, k] = tensor_hat.copy()

    return tensor_hat
```

> Hope you have fun if you work with our code!

- Competing Models

  - [Tmac-TT (Tensor Train)](https://nbviewer.jupyter.org/github/xinychen/geotensor/blob/master/Tmac-TT.ipynb)
  - [HaLRTC](https://nbviewer.jupyter.org/github/xinychen/geotensor/blob/master/HaLRTC.ipynb)

- Inpainting Examples (by **`GLTC-Geman`**)

<table>
  <tr>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_mar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_mar.jpg?size=150" width="150px;" alt="Missing at random (MAR)"/><br /><sub><b>Missing at random (MAR)</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_rmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_rmar.jpg?size=150" width="150px;" alt="Row-wise MAR"/><br /><sub><b>Row-wise MAR</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_cmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_cmar.jpg?size=150" width="150px;" alt="Column-wise MAR"/><br /><sub><b>Column-wise MAR</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/lena_rcmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/lena_rcmar.jpg?size=150" width="150px;" alt="(Row, column)-wise MAR"/><br /><sub><b>(Row, column)-wise MAR</b></sub></a><br /></td>
  </tr>
  <tr>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_mar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_mar.jpg?size=150" width="150px;" alt="RSE = 6.74%"/><br /><sub><b>RSE = 6.74%</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_rmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_rmar.jpg?size=150" width="150px;" alt="RSE = 8.20%"/><br /><sub><b>RSE = 8.20%</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_cmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_cmar.jpg?size=150" width="150px;" alt="RSE = 10.80%"/><br /><sub><b>RSE = 10.80%</b></sub></a><br /></td>
    <td align="center"><a href="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_rcmar.jpg"><img src="https://github.com/xinychen/geotensor/blob/master/data/GLTC_Geman_lena_rcmar.jpg?size=150" width="150px;" alt="RSE = 8.38%"/><br /><sub><b>RSE = 8.38%</b></sub></a><br /></td>
  </tr>
</table>


## Reference

- **General Matrix/Tensor Completion**

| No | Title | Year | PDF | Code |
|:--|:------|:----:|:---:|-----:|
|  1 | Tensor Completion for Estimating Missing Values in Visual Data | 2013 | [TPAMI](https://doi.org/10.1109/TPAMI.2012.39) | - |
|  2 | Efficient tensor completion for color image and video recovery: Low-rank tensor train | 2016 | [arxiv](https://arxiv.org/pdf/1606.01500.pdf) | - |
|  3 | Geometric Matrix Completion with Recurrent Multi-Graph Neural Networks | 2017 | [NeurIPS](https://arxiv.org/abs/1704.06803)| [Python](https://github.com/fmonti/mgcnn) |
|  4 | Spatio-Temporal Signal Recovery Based on Low Rank and Differential Smoothness | 2018 | [IEEE](https://doi.org/10.1109/TSP.2018.2875886) | - |

- **Tensor Train Decomposition**

| No | Title | Year | PDF | Code |
|:--|:------|:----:|:---:|-----:|
| 1 | Math Lecture 671: Tensor Train decomposition methods | 2016 | [slide](http://www-personal.umich.edu/~coronae/Talk_UM_TT_lecture1.pdf) | - |
| 2 | Introduction to the Tensor Train Decomposition and Its Applications in Machine Learning | 2016 | [slide](https://bayesgroup.github.io/team/arodomanov/tt_hse16_slides.pdf) | - |

- **Matrix/Tensor Completion + Nonconvex Regularization**

| No | Title | Year | PDF | Code |
|:--|:------|:----:|:---:|-----:|
| 1 | Generalized  noncon-vex nonsmooth low-rank minimization | 2014 | [CVPR](https://doi.org/10.1109/CVPR.2014.526) | [Matlab](https://github.com/canyilu/IRNN) |
| 2 | Generalized Singular Value Thresholding | 2015 | [AAAI](https://arxiv.org/abs/1412.2231) | - |
| 3 | Scalable Tensor Completion with Nonconvex Regularization | 2018 | [arxiv](http://arxiv.org/pdf/1807.08725v1.pdf) | - |
| 4 | Large-Scale Low-Rank Matrix Learning with Nonconvex Regularizers | 2018 | [TPAMI](https://ieeexplore.ieee.org/document/8416722/) | - |
| 5 | Nonconvex Robust Low-rank Matrix Recovery | 2018 | [arxiv](https://arxiv.org/pdf/1809.09237.pdf) | [Matlab](https://github.com/lixiao0982/Nonconvex-Robust-Low-rank-Matrix-Recovery) |
| 6 | Matrix Completion via Nonconvex Regularization: Convergence of the Proximal Gradient Algorithm | 2019 | [arxiv](http://arxiv.org/pdf/1903.00702v1.pdf) | [Matlab](https://github.com/FWen/nmc) |
| 7 | Efficient Nonconvex Regularized Tensor Completion with Structure-aware Proximal Iterations | 2019 | [ICML](http://proceedings.mlr.press/v97/yao19a/yao19a.pdf) | [Matlab](https://github.com/quanmingyao/FasTer) |
| 8 | Guaranteed Matrix Completion under Multiple Linear Transformations | 2019 | [CVPR](http://openaccess.thecvf.com/content_CVPR_2019/papers/Li_Guaranteed_Matrix_Completion_Under_Multiple_Linear_Transformations_CVPR_2019_paper.pdf) | - |

- **Rank Approximation + Nonconvex Regularization**

| No | Title | Year | PDF | Code |
|:--|:------|:----:|:---:|-----:|
|  1 | A General Iterative Shrinkage and Thresholding Algorithm for Non-convex Regularized Optimization Problems | 2013 | [ICML](http://proceedings.mlr.press/v28/gong13a.pdf) | - |
|  2 | Rank Minimization with Structured Data Patterns | 2014 | [ECCV](http://www1.maths.lth.se/matematiklth/vision/publdb/reports/pdf/larsson-olsson-etal-eccv-14.pdf) | - |
|  3 | Minimizing the Maximal Rank | 2016 | [CVPR](http://www1.maths.lth.se/matematiklth/vision/publdb/reports/pdf/bylow-olsson-etal-cvpr-16.pdf) | - |
|  4 | Convex Low Rank Approximation | 2016 | [IJCV](http://www.maths.lth.se/vision/publdb/reports/pdf/larsson-olsson-ijcv-16.pdf) | - |
|  5 | Non-Convex Rank/Sparsity Regularization and Local Minima | 2017 | [ICCV](http://openaccess.thecvf.com/content_ICCV_2017/papers/Olsson_Non-Convex_RankSparsity_Regularization_ICCV_2017_paper.pdf), [Supp](http://openaccess.thecvf.com/content_ICCV_2017/supplemental/Olsson_Non-Convex_RankSparsity_Regularization_ICCV_2017_supplemental.pdf) | - |
|  6 | A Non-Convex Relaxation for Fixed-Rank Approximation | 2017 | [ICCV](http://openaccess.thecvf.com/content_ICCV_2017_workshops/papers/w25/Olsson_A_Non-Convex_Relaxation_ICCV_2017_paper.pdf) | - |
|  7 | Inexact Proximal Gradient Methods for Non-Convex and Non-Smooth Optimization | 2018 | [AAAI](http://www.pitt.edu/~zhh39/others/aaaigu18a.pdf) | - |
|  8 | Non-Convex Relaxations for Rank Regularization | 2019 | [slide](https://icerm.brown.edu/materials/Slides/sp-s19-w3/Non-Convex_Relaxations_for_Rank_Regularization_]_Carl_Olsson,_Chalmers_University_of_Technology_and_Lund_University.pdf) | - |
|  9 | Geometry and Regularization in Nonconvex Low-Rank Estimation | 2019 | [slide](http://users.ece.cmu.edu/~yuejiec/papers/NonconvexLowrank.pdf) | - |

## Our Publication

**Geometric low-rank tensor completion for color image inpainting**. [coming soon!]

> Please consider citing our paper if it helps your research.

Collaborators
--------------

<table>
  <tr>
    <td align="center"><a href="https://github.com/xinychen"><img src="https://github.com/xinychen.png?size=80" width="80px;" alt="Xinyu Chen"/><br /><sub><b>Xinyu Chen</b></sub></a><br /><a href="https://github.com/xinychen/geotensor/commits?author=xinychen" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/Vadermit"><img src="https://github.com/Vadermit.png?size=80" width="80px;" alt="Jinming Yang"/><br /><sub><b>Jinming Yang</b></sub></a><br /><a href="https://github.com/xinychen/geotensor/commits?author=Vadermit" title="Code">💻</a></td>
  </tr>
</table>

See the list of [contributors](https://github.com/xinychen/geotensor/graphs/contributors) who participated in this project.


License
--------------

This work is released under the MIT license.

