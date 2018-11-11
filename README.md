# BundleFusion
     2017年斯坦福大学提出的BundleFusion算法，可以说是目前基于RGB-D相机进行稠密三维重建效果最好的方法了。
 
![BundleFusion](img/teaser.jpg)


You are free to use this code with proper attribution in non-commercial applications (Please see [License.txt](License.txt)).

More information about this project can be found in our [paper](https://arxiv.org/pdf/1604.01093.pdf) and [project website](http://graphics.stanford.edu/projects/bundlefusion/).

# 简介
     BundleFusion 和之前所有的帧匹配（keyframe，每个 chunk 的第一帧），
     匹配采用 sift 描述子，这种方式和 SLAM 中一般通过邻近的帧计算 pose 不一样。

     为了减少优化时优化的变量个数，采用 local-to-global 的位子优化策略。
     local 范围内，将全部帧按照划分成等大小的 chunk，先在 chunk 内做 pose 优化。
     然后在 global 范围内，所有的 chunk 的 pose 放在一起优化。
     这种 local-to-global 的方式，可以减少优化的变量个数，加速优化过程。
     在建图部分，采用 voxel hashing 算法，不同的是，BundleFusion 中每个 chunk 的 pose 一直在做改变，
     pose 的改变需要映射到全局的 TSDF 模型中。方法是，挑选出 pose 改变量最大的 chunks（10 个），
     按照将帧融合到 TSDF 时的 pose，做 de-integration，将之前融合进 TSDF 模型中的数据，从 TSDF 模型中减掉。
     然后，按照优化之后的 pose，将 chunk 的数据 re-integrate 到 TSDF 模型中。
 
     算法使用局部小块优化和全局优化解决位姿求解的 drift 和回环检测问题，
     并且用 integration和de-integration的方式解决位姿优化后重建场景更新的问题。

# 解析
[BundleFusion 解析](https://blog.csdn.net/fuxingyin/article/details/52921958)

     输入：
          RGB-D相机采集的对齐好的color+depth的数据流
     输出:
          重建好的三维场景。
     步骤:
          1. 相关关系搜索
               这里使用的是一种sparse-then-dense的并行全局优化方法。
               特征检测、特征匹配、相关关系验证;
               a. 先使用稀疏的 SIFT特征点 来进行比较粗糙的配准，
                  因为稀疏特征点本身就可以用来做loop closure检测和relocalization。
               b. 然后使用稠密的几何和光度连续性进行更加细致的配准。
               
               特征点对需要经历如下三种考验才能通关：
                   第一种是直接对关键点本身分布的一致性和稳定性进行考验。(根据位置关系等剔除误匹配)
                   第二道关卡是对特征匹配对跨越的表面面积进行考验，去掉特别小的，因为跨越面积较小的的话很容易产生歧义。
                   第三道关卡是进行稠密的双边几何和光度验证，去掉重投影误差较大的匹配对。
                   
               对于新获取的帧，和之前的帧累计到一个 chunk 时（每 11 帧组成一个 chunk），
               先在 chunk 内匹配，做 local 优化，优化后，用 chunk 内的第一帧表示 chunk，
               chunk 内的所有帧的特征组合在一起，表示该 chunk 内特征，然后新的 chunk 和之前所有的 chunk 匹配，
               做 pose 优化，SIFT 特征提取和匹配都在 GPU 上做，计算一帧 SIFT 关键点和提取关键点描述子占用 4-5ms，
               匹配一次耗时大概 0.05 ms，特征匹配不可避免会有错误的匹配，作者设计了严格的筛选机制。
               
          2. 位姿优化
             这里使用了一种分层的 local-to-global 的优化方法。
             因为可以剥离出关键帧，减少存储和待处理的数据。
             并且这种分层优化方法减少了每次优化时的未知量，
             保证该方法可扩展到大场景而漂移很小。
             
             a. 局部位姿优化 GN球解器 高斯牛顿法
               在最低的第一层，每连续10帧组成一个chunk，第
               一帧作为关键帧，然后对这个chunk内所有帧做一个局部位姿优化。
             b. 全局位姿优化 GN球解器 高斯牛顿法
               在第二层，只使用所有的chunk的关键帧进行互相关联然后进行全局优化。
               
          3. 动态场景更新
               系统中，global pose 优化会改变所有帧的 pose，
               作者使用 de-integration 和 re-integration 根据 pose 的变化，
               更新 TSDF 值。TSDF 模型表示采用 voxel hashing 的形式，
               RGB-D 数据的  加操作(integration ) 和 减操作(de-integration) 是对称的，
               integrate 的数据，可以按照 integrate 时的 pose，
               从模型中做 de-integration 减掉。
               
               作者对于优化的位姿按照位姿的变化的大小排序，
               将 pose 变化最大的 10 帧数据先按照优化前的 pose 做de-integration，
               从模型中减掉，然后再按照优化后的 pose 做 re-integration。
              
              动态3D重建
                    这里的dynamic 3D reconstruction指的是根据优化的pose调整之前重建好的部分。
                    这里对于TSDF模型的更新不仅有 加操作(integration )，还有 减操作(de-integration)。
                    
     关于运行速度，目前可以做到在两块GPU（GTX Titan X + GTX Titan Black）上实时。
     在演示视频中，structure sensor是卡在iPad上，
     将采集到的数据通过无线网络传给台式机（带GPU），
     匹配、优化和重建工作都是在台式机上运行，
     重建的结果再通过无线网络传到iPad上显示。
     
     问题及改进方向
          由于成像传感器存在噪音，稀疏关键点匹配可能产生小的局部误匹配。
               这些误匹配可能会在全局优化中传播，导致误差累积。
          上述效果图片都是在作者提供的公开数据集上的效果，该数据集采集的场景纹理比较丰富，光照也比较适中。
          而实际重建时效果和所使用深度相机的性能、待重建场景的纹理丰富程度关系很大。
          对于办公室这种简洁风格的场景效果会下降很多，还有很多可改进的地方。
          目前算法需要两块GPU才能实时运行，算法的优化和加速非常有意义。
          
          特征坐标在优化时是保持不变的，如果 SIFT 特征偏移几个像素，或者深度图像中特征位置处噪声比较大，
          这种误差会造成最小二乘求解的 SIFT 特征点的位置不对，从而优化的 pose 有误差。
          理想情况下，系统应该做 BA 优化，把特征的坐标也放在优化中，但是，这样计算量会很大。
          
          实测 BundleFusion 在重复结构纹理的时候，效果不是特别好；
          建立的三维结构可能因为上述原因，特征的位置不对，使得重建的三维结构也可能会有偏差。



## Installation
The code was developed under VS2013, and tested with a Structure Sensor.

Requirements:
- DirectX SDK June 2010
- NVIDIA CUDA 7.0
- our research library mLib, a git submodule in external/mLib
- mLib external libraries can be downloaded [here](http://kaldir.vc.in.tum.de/mLib/mLibExternal.zip)

Optional:
- Kinect SDK (2.0 and above)
- Prime sense SDK


## Citation:  
```
@article{dai2017bundlefusion,
  title={BundleFusion: Real-time Globally Consistent 3D Reconstruction using On-the-fly Surface Re-integration},
  author={Dai, Angela and Nie{\ss}ner, Matthias and Zoll{\"o}fer, Michael and Izadi, Shahram and Theobalt, Christian},
  journal={ACM Transactions on Graphics 2017 (TOG)},
  year={2017}
}
```

## Contact:
If you have any questions, please email Angela Dai at adai@cs.stanford.edu.



We are also looking for active participation in this open source effort making large-scale 3D scanning publically accessible. Please contact us :)
