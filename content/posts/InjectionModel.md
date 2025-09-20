---
title: "OpenFOAM 自定义injection model"
date: 2025-09-16
draft: false
tags: ["spray","injection model","lagrangian"]
categories: ["src"]
summary: "在SprayFoam基础上添加自定义的injection model"
cover:
  image: "images/cover/OpenFOAM_logo.png"
  alt: "atomization and sprays"
  caption: "OpenFOAM Logo"
---
# OpenFOAM 自定义injection model

> OpenFOAM版本：`OpenFOAM-v2312`

---

## 1. injection model介绍

### 喷雾子模型

使用`sprayFoam`计算燃烧喷雾流时主要通过`constant/sprayCloudProperties`来定义喷雾特征

{{< collapse summary="sprayCloudProperties示例：" >}}
<a id="patch"></a>
```c++
subModels
{
   injectionModels
   {
      model1
      {
         type            patchInjection;
         patch           nozzle;
         parcelBasisType mass;
         SOI             0;         //- Start of injection [s]
         massTotal       1e-3;      //- Total mass to inject [kg]
         duration        1;         //- Injection duration [s]
         parcelsPerSecond 1;        //- Number of parcels to introduce per second
         U0              (0 0 0);
         massFlowRate constant 0.01;
         flowRateProfile constant 0.001; //- Flow rate; (table((time flowRate)))
         sizeDistribution
         {
               type        fixedValue; //- RosinRammler & RosinRammlerDistribution
               fixedValueDistribution
               {
                  value   0.0002;
               }
         }
         // As we want 1 particle per parcel, this avoid
         // accumulated vol if nParticles is below 1
         minParticlesPerParcel   0.7;
      }
   }
   dispersionModel   none;
   atomizationModel  none;
   breakupModel      ReitzDiwakar;
   heatTransferModel RanzMarshall;
   phaseChangeModel  liquidEvaporationBoil;
   stochasticCollisionModel   trajectory;
   ...
}
```
{{< /collapse >}}

其中`injectionModels`作为`sprayCloud`的子模型来使用。

> **主要使用到的参数及其含义：**

> `SOI`：喷射开始时间  
> `massTotal`：喷射总质量  
> `duration`：喷射持续时间  
> `parcelsPerSecond`：注入 parcel 的频率（影响统计收敛），**parcel里可以有多个particles**  
> `parcelBasisType`：注入液滴方法（mass/fixed），mass基于`massTotal`计算nParticle，fixed则需要给定nParticle。  
> > 使用fixed时，虽然不会用到`massTotal`,但是最好给定一个非0常数，因为sprayCloud构造函数初始化时仍会调用`averageParcelMass(){ return mass/massTotal; }`会报错。  
>
> `flowRateProfile`：喷射速率随时间的分布（可设为 constant，或通过表格定义）  
> `position` & `direction`：喷口位置和方向  
> `thetaInner` & `thetaOuter`：半锥内角和外角，修改`thetaInner`来定义空心或实心喷雾
> `innerDiameter` & `outerDiameter`：喷嘴内外直径  
> `d0` & `U0`：液滴初始直径和速度  
> `sizeDistribution`：液滴直径分布（uniform/normal/rosinRammler 等）

还有一些其他的参数，没有一一列出，细节可查看[OpenFOAM源代码](https://develop.openfoam.com/Development/openfoam/-/tree/master/src/lagrangian/intermediate/submodels/Kinematic/InjectionModel?ref_type=heads)。

### sprayCloud中的InjectionModel类

{{< figure src="/openfoam-feifan/images/posts/injectionModels.png" caption="OpenFOAM的InjectionModel模板类" align="center" >}}

`sprayCloud`的`InjectionModel`模板类的预定义路径为：`$FOAM_SRC/lagrangian/spray/parcels/include/makeSprayParcelInjectionModels.H`（**很重要**），主要有11种。

- coneNozzleInjection

    - 作用：实际喷嘴建模，考虑喷孔直径、锥角，质量流量随时间演变。
    - 参数：喷孔直径、喷雾角、初始液滴直径、流量曲线等
    - 场景：柴油机、内燃机喷嘴。

 - coneInjection

    - 作用：从指定点以锥形角度喷入颗粒/液滴。
    - 参数：喷射点、喷射方向、锥角、速度、流量。
    - 场景：模拟燃油喷嘴的理想锥形喷射。（Multi-point cone injection model）

 - FieldActivatedInjection

    - 基于流场或标量场条件触发注入
    - For injection to be allowed `factor*referenceField[celli] >= thresholdField[celli]`, where:
        - referenceField is the field used to supply the look-up values
        - thresholdField supplies the values beyond which the injection is
            permitted.
    - Limited to a user-supplied number of injections per injector location

 - InjectedParticleDistributionInjection

    - 从已有数据或外部粒子分布文件`cloud`读取喷嘴器，将原始粒子转化为自定义`bin width`的粒径分布。
    - 通过打开`applyDistributionMassTotal`重新设置输入液滴总质量。
    - 每个喷注位置使用恒定流量曲线。
    - 以上为英文翻译，仅供参考，tutorials中没有具体实例。

 - InjectedParticleInjection

    从粒子列表注入液滴，每个`parcel`对应 一个粒子。适合复制已有实验或前一次仿真数据。
    ```c++
    model1
    {
        type            injectedParticleInjection;
        SOI             0;
        massTotal       0; // Place holder only
        parcelBasisType fixed;
        nParticle       1; // 1 particle per parcel
        cloud           eulerianParticleCloud;
        positionOffset  (-0.025 2 -0.025);
    }
    ```

 - InflationInjection
    - 将液滴注入到 特定网格层或边界附近，通常配合`mesh refinement`或边界膨胀层使用。
    - creates new particles by splitting existing particles within in a set of generation cells, then inflating them to a target diameter within the generation cells and an additional set of inflation cells.


 - manualInjection

    - 作用：完全用户定义，直接在字典里给出每个 parcel 的位置、速度、大小。
    - 参数：每个 parcel 的坐标、速度、直径。**仅在SOI定义喷雾空间分布，并不是持续的喷雾。**
    - 场景：验证性算例，或重现实验测得的初始分布。

 - NoInjection

    - 作用：空模型，不做任何注入。
    - 参数：无。
    - 场景：测试、只跟踪已有云粒子时使用。

 - patchInjection

    - 作用：从指定 patch 面（边界）注入颗粒，见[上述示例](#patch)。
    - 参数：patch 名称、注入速度、流量。
    - 场景：壁面喷嘴、边界面源项。液滴随机分布在`patch`上。

 - PatchFlowRateInjection
    
    - 按 patch 流量分配，parcel 分布更符合实际流场，可模拟非均匀入口流场下的喷雾注入
    - Initial parcel velocity given by local flow velocity
   
---

## 2. injection model内部调用逻辑

### 几个重要文件

> `$injectionModel`=`$FOAM_SRC/lagrangian/intermediate/submodels/Kinematic/InjectionModel`
> `$make***InjectionModel`=`$FOAM_SRC/lagrangian/intermediate/particels/include/make***ParcelInjectionModels.H`
> `$sprayMake`=`$FOAM_SRC/lagrangian/spray/parcels/include/makeSprayParcelInjectionModels.H`

 - `$injectionModel`定义了`InjectionModel`模板类的声明以及全部的injection models的实现。`InjectionModel`模板类的声明：
   ```c++
   namespace Foam
   {
      template<class CloudType>
      class InjectionModel
      :
         public CloudSubModelBase<CloudType>
      {}
   }
   ```
 - `$make***InjectionModel`实现了各种`parcel`的`InjectionModel`注册。基础粒子在`***=Parcel`中注册，而其他更复杂的`parcel`类型如`ReactingParcel`,`ReactingMulitphaseParcel`等则在其他H文件中注册。**不同H文件中加载的`InjectionModel`类型和数量会有不同**。注册通过宏定义语句实现，定义在`InjectionModel.H`中：
 
   {{< collapse summary="宏定义注册代码" >}}
   ```c++
   #define makeInjectionModelType(SS, CloudType)                                  \
                                                                                 \
      typedef Foam::CloudType::kinematicCloudType kinematicCloudType;            \
      defineNamedTemplateTypeNameAndDebug(Foam::SS<kinematicCloudType>, 0);      \
                                                                                 \
      Foam::InjectionModel<kinematicCloudType>::                                 \
         adddictionaryConstructorToTable<Foam::SS<kinematicCloudType>>          \
               add##SS##CloudType##kinematicCloudType##ConstructorToTable_;
   ```
   {{< /collapse >}}
 - > 通过上述注册，将数据保存在静态变量中，用`dictionaryConstructorTablePtr_`指针调用。针对不同`CouldType`创建了不同的哈希表，并基于求解器使用的云类来查找子模型。
 - `$sprayMake`才是`sprayFoam`实现`InjectionModel`注册的地方，共有11个injection models。

### 调用逻辑

`injection model`通过[`run time selector(RTS)`](https://xiaopingqiu.github.io/2016/03/12/RTS1/)机制实现运行时选择。分为几个流程:
 1. 粒子库编译：`$make***InjectionModel`通过哈希表的形式保存在`liblagrangianIntermediate`动态库中。基于`solver`选择的不同`CloudType`来调用不同的H文件；再基于`sprayCouldProperties`中`injectionModel.type`名称来动态选择不同的`InjectionModel`类。**尽管很多类型，但是实际上这个动态库中的哈希表并没有被`sprayFoam`调用**。
 2. 喷雾库编译：`$sprayMake`的子模型通过哈希表的形式保存在`liblagrangianSpray.so`动态库中。此时`CloudType`为`basicSprayCloud`;`injectionModel.type`有11种选择，在上节有介绍到。
    > 可以通过`nm -d liblagrangianSpray.so | grep ManualInjection`查看是否已添加该模型。
 3. 求解器初始化：在`sprayFoam`中调用`createClouds.H`时，构造函数会自动初始化他的各种`submodels`，此时的模板类`InjectionModel`变成了`patchInjection<basicSprayCloud>`。

使用`gdb`难以对injection model进行调试，因为在编译过程中各种类型名称对应的类都已经写入了哈希表，并在构造函数的实现中（`InjectionModelNew.C`）通过以下语句调用：
   ```c++
   auto* ctorPtr = dictionaryConstructorTable(modelType);
   ```
若调用失败，则返回`sprayCloud`的所有`InjcetionModels`。

---

## 3. 自定义射流子模型

由于OpenFOAM自带的11个模型中没有可以同时自定义喷注位置、粒径、流量的`InjectionModel`。因此，希望通过写一个更通用的自定义喷注模型，用于横向液体射流计算。模型可以输入更复杂，但是自定义的功能得更强大。

对`InjectionModel`调用逻辑有了更清楚的认识后，基于`RTS`机制可以很轻松的写一个新的喷射模型`myInjectionModel`。

 - 模型基于`ManualInjection`作为基础，在该模型上增加射流流量输入。
 - 模型需要使用`makeInjectionModelType`函数来将其注册到`basicSprayCloud`的哈希表中。仅需几行代码即可实现：
   ```C++
   #include "basicSprayCloud.H"
   #include "MyManualInjection.H"

   makeInjectionModelType(MyManualInjection, basicSprayCloud);
   ```
   相较于`$injectionModel`中模型的`option`文件，需要添加`intermediate`和`spray`相关的库以调用`makeInjectionModelType`函数和`basicSprayCloud`类。
 - 计算时在`controlDict`中加入:
   ```c++
   libs     ( "libmyInjectionModels.so" );
   ```

源代码可查看[github链接](https://github.com/feifan142/deformSprayFoam)。

---

## 4. 教程算例

