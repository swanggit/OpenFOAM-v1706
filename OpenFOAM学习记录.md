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

# 1 Basic

包括三个求解器：

- **laplacianFoam**：求解器简单的拉普拉斯方程。如固体中的热传导方程。
- **potentialFoam**：势流求解器。求解速度势。
- **scalarTransportFoam**：求解稳态或瞬态的标量传输方程

# 2 Incompressible（不可压缩流动）

- **adjointShapeOptimizationFoam**：不可压缩非牛顿流体湍流流动稳态求解器，此求解器包括根据压力损失进行管道形状优化功能。
- **boundaryFoam**：不可压缩稳态一维湍流求解器。常用于为入口边界生产边界层条件
- **icoFoam**：牛顿流体不可压缩瞬态层流求解器
- **nonNewtonianIcoFoam**：非牛顿流体不可压缩瞬态层流求解器
- **pimpleFoam**：采用PIMPLE算法的大时间步不可压缩湍流瞬态求解器
- **pimpleDyMFoam**：用于运动网格的牛顿流体不可压缩湍流瞬态求解器
- **pisoFoam**：使用PISO算法的不可压缩湍流瞬态求解器
- **shallowWaterFoam**：瞬态无粘有旋浅水方程瞬态求解器
- **simpleFoam**：使用SIMPLE算法的不可压缩湍流稳态求解器
- **porousSimpleFoam**：支持MRF的多孔介质不可压缩湍流稳态求解器
- **SRFSimpleFoam**：单参考系中非牛顿流体不可压缩湍流稳态求解器

# 3 Compressible（可压缩流动）

- **rhoCentralFoam**：基于Kurganov&Tadmor中心迎风格式的密度计可压缩湍流求解器
- **rhoCentralDyMFoam**：支持动网格的基于Kurganov&Tadmor中心迎风格式的密度计可压缩湍流求解器
- **rhoPimpleFoam**：基于PIMPLE算法的可压缩湍流瞬态求解器，常用于HVAC领域
- **rhoPimpleDyMFoam**：与rhoPimpleFoam相同，不过附加了动网格求解
- **rhoPorousSimpleFoam**：附加有多孔介质模型的可压缩湍流稳态求解器
- **sonicFoam**：瞬态可压缩气体湍流求解器，用于跨音速和超音速
- **sonicDyMFoam**：与sonicFoam相同，可以使用动网格
- **sonicLiquidFoam**：可压缩跨音速/超音速层流瞬态求解器

# 4 Multiphase（多相流）

- **cavitatingFoam**：基于均相平衡模型瞬态空化求解器。
- **cavitatingDyMFoam**：与cavitatingFoam相同，支持动网格及自适应网格
- **compressibleInterFoam**：基于VOF模型的可压缩、非等温、不可溶两相界面捕捉求解器
- **compressibleInterDyMFoam**：与compressibleInterFoam功能相同，支持动网格与自适应网格
- **compressibleMultiphaseInterPhase**：基于VOF模型的支持n相不可压、非等温、不可溶流体界面捕捉求解器
- **driftFluxFoam**：基于mixture模型，考虑相间滑移的两相不可压缩求解器
- **interFoam**：基于VOF模型的两相不可压缩、等温、不可溶流体界面捕捉求解器
- **interDyMFoam**：与interFoam功能相同，支持动网格及自适应网格
- **interMixingFoam**：三相不可压缩，其中两相互溶，使用VOF模型捕捉相间界面
- **interPhaseChangeFoam**：基于VOF模型的不可压、等温、不可溶、存在相变的两相界面捕捉求解器
- **interPhaseChangeDyMFoam**：与interPhaseChangeFoam功能相同，支持动网格及自适应网格
- **multiphaseEulerFoam**：包含传热的多相可压缩求解器，基于双流体模型
- **multiphaseInterFoam**：考虑表面张力及接触角效应的多相不可压界面捕捉求解器
- **multiphaseInterDyMFoam**：与multiphaseInterFoam功能相同，支持动网格及自适应网格
- **potentialFreeSurfaceFoam**：包含波高的不可压缩NS方程求解器，可用于模拟单相自由表面的波高
- **potentialFreeSurfaceDyMFoam**：与potentialFreeSurfaceFoam功能相同，支持动网格及自适应网格
- **reactingMultiphaseEulerFoam**：用于具有共同压力的任何数量的可压缩流体相的系统，但是另外具有分离的性质。 相模型的类型在运行时选择，并且可以有选择地表示多个组分反应。
- **reactingTwoPhaseEulerFoam**：用于具有共同压力的2相可压缩流体系统，但是另外具有分离的性质。 相模型的类型在运行时选择，并且可以有选择地表示多个物种和同相反应。
- **twoLiquidMixingFoam**：不可压缩可溶两相混合求解器
- **twoPhaseEulerFoam**：两相可压缩系统，其中一相为分散相。典型应用为包含传热模型的流体中的气泡。

# 5 直接数值模拟

- **dnsFoam**：计算域为立方体的各向同性湍流直接模拟求解器

# 6 Combustion（燃烧）

- **chemFoam**：单网格化学反应求解器。主要用于与其他化学反应求解器作对比。
- **coldEngineFoam**：内燃机内冷态流动求解器
- **engineFoam**：内燃机求解器
- **fireFoam**：瞬态火灾及湍流扩散火焰求解器
- **PDRFoam：附带湍流的可压预混/部分预混燃烧求解器**
- **reactingFoam**：附带化学反应的燃烧求解器
- **rhoReactingBuoyantFoam**：利用密度基、热力学模型及浮力强化模型的燃烧求解器，包含化学反应
- **rhoReactingFoam**：密度基、热力学模型及化学反应的燃烧求解器
- **XiFoam**：包含湍流模型的可压缩预混/部分预混燃烧求解器

# 7 Heat Transfer（传热）

- **buoyantBoussinesqPimpleFoam**：包含湍流的瞬态不可压浮力驱动求解器
- **buoyantBoussinesqSimpleFoam**：包含湍流的稳态不可压浮力驱动求解器
- **buoyantPimpleFoam**：包含湍流的瞬态可压缩浮力驱动求解器（用于暖通和传热）
- **buoyantSimpleFoam**：包含湍流的稳态可压缩浮力驱动求解器
- **chtMultiRegionFoam**：固体热传导与流体传热耦合求解，包含浮力、湍流以及热传导瞬态求解器
- **chtMultiRegionSimpleFoam**：固体热传导与流体传热耦合求解，包含浮力、湍流以及热传导稳态求解器
- **thermoFoam**：在固定流程中的能量传输及热力学求解器

# 8 Particle-tracking（粒子跟踪）

- **coalChemistryFoam**：包含煤粉及石灰岩颗粒能量源及燃烧的可压缩、湍流瞬态求解器
- **DPMFoam**：考虑颗粒体积分数对连续相作用的单颗粒团耦合输运瞬态求解器
- **MPPICFoam**：基于MPPIC方法描述颗粒间的碰撞，不真实求解颗粒与颗粒的相互作用
- **icoUncoupledKinematicParcelFoam**：单颗粒团被动输运瞬态求解器
- **icoUncoupledKinematicParcelFoam**：支持动网格及自适应网格的icoUncoupledKinematicParcelFoam
- **reactingParcelFilmFoam**：包含化学反应、多项颗粒以及壁膜模型的可压缩湍流求解器
- **reactingParcelFoam**：包含化学反应、多相颗粒以及可选的源/约束的可压缩湍流瞬态求解器
- **simpleReactingParcelFoam**：包含化学反应、多相颗粒以及可选的源及约束的可压缩湍流稳态求解器
- **sprayFoam**：喷雾颗粒可压缩湍流瞬态求解器
- **sprayDyMFoam**：支持动网格和自适应网格的喷雾颗粒可压缩湍流瞬态求解器
- **sprayEngineFoam**：引擎喷雾颗粒可压缩湍流瞬态求解器
- **uncoupledKinematicParcelFoam**：颗粒团单相耦合瞬态求解器

# 9 Electromagnetics（电磁场）

- **electrostaticFoam**：静电场求解器
- **magneticFoam**：永磁场求解器
- **mhdFoam**：不可压缩层流磁流体求解器

# 10 Stress analysis of solid（固体应力）

- **solidDisplacementFoam**：线弹性小应变的有限体积瞬态分离式求解器，可选温度扩散和热应力
- **solidEquilibriumDisplacementFoam**：线弹性小应变的有限体积稳态分离式求解器，可选温度扩散和热应力

# 11 Finance（金融）

- **financialFoam**：期权定价Black0Scholes方程求解器



# TDAC 

TDAC是一种结合了原位自适应制表法(ISAT)和动态自适应化学的算法(DAC),在传统的DAC中，在每个时间步和每个网格点上进行机理简化过程，生成考虑当地和当前热力学条件和物种组成的最优简化机理。然而，机理简化过程的计算费用昂贵，特别是在网格点数量较多的情况下。在TDAC中，这个问题由ISAT技术解决，只有当前和局部状态不在表中化学的精确范围内时，才需要机理简化。在ISAT中，这一精度区域是量化的，并动态增加表格

TDAC的另一个重要特征是对物种输运方程的简化。当产生简化机制并剔除不重要物种时，只求解重要物种的输运方程，进一步降低了计算成本

# Sutherland**粘度**

Sutherland公式通过两个模型参数As和Ts计算动态粘度μ。

 ![](https://oscimg.oschina.net/oscnet/3368c77e-e494-483c-a382-da4f3a15aa99.png) 

![](https://oscimg.oschina.net/oscnet/e5aec003-4e51-4293-921f-cdfab770271f.png)

# **热导率**

对于导热系数，我们使用了修正的Eucken相关，它从粘度和其他性质计算导热系数。这是可以做到的，因为导热系数的温度依赖性与粘度的温度依赖性大致相同

![img](https://oscimg.oschina.net/oscnet/207d37b8-2274-4a93-95cb-59c21882713b.png)



# **thermo**

thermo模型处理比热和其他派生属性的计算。

JANAF这个名字源于美国陆海空联合热化学工作组，该工作组自20世纪60年代起就发表了许多化合物的热力学性质。我们也发现了对JANNAF的引用，这就把NASA也带入了缩写。此外，在JANAE表下的多项式也被称为NASA多项式或7参数NASA多项式。JANAF模型是OpenFOAM提供的thermo模型之一。JANAF模型依赖于两组模型系数。当温度低于阈值Tcommon时，采用lowCpCoeffs;当温度较高时，采用highCpCoeffs系数。从数学上讲，JANAF模型实现了以下等式:

![img](https://oscimg.oschina.net/oscnet/61b3e348-0b68-4b48-8ad4-cc4f9da0e074.png)

![img](https://oscimg.oschina.net/oscnet/51974e36-a5bb-4d24-9635-fc197397ede9.png)

JANAF模型系数是两组7个系数。使用两个多项式可以覆盖较宽的温度范围。因此，第一个多项式通常覆盖低温区域，200-1000k，而第二个多项式覆盖高温区域，1000-5000 K。这两个多项式被约束以产生1000 K的值。此外，低温区域的多项式被固定为298.15 K，结合1bar的压力作为标准参考状态。将多项式固定到标准状态可以保证，在标准状态时热力学性质得到精确的再现。

一个多项式的前5个系数用于计算比热容Cp，剩余两个系数用于计算焓和熵。

# chemkin

分子结构计算粘度系数、导热系数和扩散系数

# k-epsilon模型初始值

k的边界条件可以用turbulentIntensityKineticEnergyInlet来给，这样你只需要给湍流强度，那个value的值只是一个place holder，给什么都可以。epsilon也有一个对应的turbulentMixingLengthDissipationRateInlet，这里就需要你给定一个长度尺度了（比如特征长度的7%）。

![k = \frac{3}{2} \; (U \, I)^2](https://www.cfd-online.com/W/images/math/5/a/e/5ae4d30823f9b99be3b0c06d6e48abb8.png)

![\epsilon = C_\mu^\frac{3}{4} \, \frac{k^\frac{3}{2}}{l}](https://www.cfd-online.com/W/images/math/b/e/9/be9124ef78e213bd751ce232c0a99d21.png)

Where ![C_\mu](https://www.cfd-online.com/W/images/math/5/4/e/54e955fc6e041a86bbe25574d4119bcd.png) is a turbulence model constant which usually has a value of ![0.09](https://www.cfd-online.com/W/images/math/3/0/5/305cf1fb13b9539dcd317a0354c9ed61.png)

