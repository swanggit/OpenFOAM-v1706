[TOC]



### OpenFOAM边界类型

边界首先被分解为一系列的*patch*，一个*patch*包含一个或多个闭合边界面，但是不一定在物理上相连。

***Base*型**： 这种类型的边界条件仅仅是针对几何而言的，或者是描述了边界数据是怎么连接的。 

***Primitive*型**：在*Base*型的基础上附带了某个场变量。

***Derived*型**：复杂的边界条件，从*Primitive*型派生出来，附带有场变量。

![img](https://img-blog.csdn.net/20140317105259187?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveHh5aGp5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

***empty***：由于***OpenFOAM***总是产生三维几何体，但是也可以用来解决一维或者二维的问题，这就需要在那些与不需要求解的维度垂直的***patch***平面上指定特殊的***empty***条件。

### **使用blockMesh生成网格**

 blockMesh从case文件夹中的constant/polyMesh文件夹中读取blockMesh字典文件并生成网格。blockMesh读取该文件后，生成网格，并将网格数据写入位于同一文件夹中*points，faces，cells，boundary*等文件中。

 **blockMesh背后的原理是将集合区域分解为一个或者更多个三维区域，即六面体区块。区块的边可以是直线，弧或者样条。从表面上看，网格就是就是在区块各个方向上指定一系列体元，需要为blockMesh提供足够的信息来产生网格数据。**

| 关键字              | 描述                                      | 实例                                                    |
| ------------------- | ----------------------------------------- | ------------------------------------------------------- |
| **convertToMeters** | **顶点坐标的比例缩放因子**                | 设置为0.001时，数值就缩放到了单位为mm                   |
| vertices            | 顶点坐标的list                            | (0, 0, 0)                                               |
| edges               | 用来描述弧形（arc）或者样条（spline）的边 | arc 1 4 (0.939 0.342 -0.5)                              |
| block               | 有序的顶点标签编号list，以及网格的尺寸    | hex(0 1 2 3 4 5 6 7)(10 10 1)simpleGrading(1.0 1.0 1.0) |
| patches             | patch的list                               | symmetryPlane base((0 1 2 3))                           |
| mergePatchPairs     | 需要合并的patch的list                     |                                                         |

| 可选关键字   | 描述                   | 需要补充的参数   |
| ------------ | ---------------------- | ---------------- |
| arc          | 使用圆弧连接           | 圆弧通过的一个点 |
| simpleSpline | 使用样条曲线连接       | 一系列内部点     |
| polyLine     | 使用一系列直线连接     | 一系列内部点     |
| polySpline   | 使用一系列样条曲线连接 | 一系列内部点     |
| line         | 直线连接               | -                |

块定义：

- 顶点编号：一般为hex六面体，其后紧跟着顶点编号的list
- 体元数量：第二条给出的块在三个方向x1、x2、x3上数量
- 体元扩展比例：给出块三个方向的体元扩展比例，扩展比例为块上该方向最后一个体元的宽度与起始宽度之比，

  **simpleGrading**: 指定在三个方向上使用均匀扩展比例，并分别指定这三个比例大小。

​      simpleGrading (1  2 3)

​    **edgeGrading**: 针对每个边界分别给出完整的cell比例，图中的箭头所示的方向就是从第一个体元到最后一个体元的方向。

​      simpleGrading (1  1 1  1  2  2  2  2  3  3  3 3)![img](https://img-blog.csdn.net/20140317105527906?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveHh5aGp5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  每个面都由4个顶点组成的list来定义，点描述的顺序必须遵循以下规则：当从块内部向外看时，描述的点的顺序是顺时针围绕面的一圈。

### OpenFOAM并行计算

拆分算域法，即将一个待解决的计算域分解成为一系列可以单独执行的离散部分，每部分用单独的处理器计算器计算，主要涉及计算域 的分解、并行计算程序、计算结果的合并。

```text
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "system";
    object      decomposeParDict;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //
numberOfSubdomains N;  //分解数，需要将目标分解域分解成n块，n<=计算器总核数，否则会报错。
method          simple;  //分解方法，主要有4种，即simple,hierarchical,scotch,manual。
simpleCoeffs       //simpleCoeffs字典，对应simple方法，简单几何分解，计算域依据依据方向被切分
{
    n               (2 2 1);  //x,y,z上的分解量。
    delta           0.001;    //网格偏斜因子。
}
hierarchicalCoeffs   //hierarchicalCoeffs字典，对应hierarchical方法，指定首先切分哪个方向
{
    n               (1 1 1); //x,y,z上的分解量。
    delta           0.001; //网格偏斜因子。
    order           xyz;  //分解顺序。
}
scotchCoeffs   //scotchCoeffs字典，对应scotch方法，用户可以为每个处理器分配不同的权重，通过processorweights关键词来指定
{
    processorWeights      (wt1 wt2 ...); //每个处理器分配的权重。
    strategy       b; //分解策略，默认为b。
}
manualCoeffs//手动分解法，用户可以直接把某一片网格区域指定给处理器，可通过setFileds进行定义
{
    dataFile        ""; //给各个处理器分配任务的字典文件名称。
}
distributed     no;//数据是否写入不同硬盘
roots           ( );//算例目录路径
// ************************************************************************* //
```

```
decomposePar//用来分解网格和场，通过读取decomposeParDict字典文件的参数，分解几何和场文件。

mpirun -np N solvername -parallel //N:并行线程数，由numberOfsubdomains决定

reconstructPar//并行计算结果重组
```

### 创建轴对称网格

![img](https://cdn.cfd.direct/docs/user-guide-v6/img/user357x.png)



1. makeAxialMesh输入：

   - 指定对称轴边界命名

   - 要分裂成两个楔形边界（前面和后面）的边界命名

2. collapseEdges

- 删除对称轴上的面

  ```c
  makeAxialMesh -case testAxial2 -axis center -wedge frontAndBackPlanes -wedgeAngle 5
  ```

