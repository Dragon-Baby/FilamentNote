# 标记

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-12-38-02-image.png)

# 材质系统

## 标准模型

材质模型可由BSDF`Bidirectional Scattering Distribution Function`描述。BSDF包含BRDF`Bidirectional Reflectance Distribution Funtion`和BTDF`Bidirectional Transmittance Function`两个部分。

本项目着重关注BRDF部分，忽略BTDF部分，但也会尽量去近似模拟相关效果。这里所介绍的标准模型会尽可能去模拟能反射光的各向异性的导电或非导电表面。

BRDF由以下两部分构成：

- 漫反射组件$f_d$

- 高光组件$f_r$

下图表述了表面、表面法线、入射光和上述两部分之间的关系

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-00-49-image.png)

上图中的光与表面反射的过程可以表示为：

                $f(v,l)=f_d(v,l)+f_r(v,l)$                 $(1)$

该等式描述了单方向入射光反射的过程，完整的渲染方程可以在半圆内积分入射光$l$得到。

入射表面通常并不平坦，这里引入微表面的概念：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-09-14-image.png)

微表面法线必须在入射光和视线方向之间才能反射可见光：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-11-01-image.png)

并不是所有符合条件的微表面可以反射光，因为BRDF会考虑遮蔽和阴影：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-12-36-image.png)

微表面BRDF受`粗糙度`影响，表面越光滑(粗糙度低)，越多的微表面会朝向合适的位置，反射光更明显，反之，反射光散射分布：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-16-38-image.png)

微表面模型等式($x$表示高光或漫反射部分)：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-17-45-image.png)

$D$描述了微表面的分布情况(NDF，法线分布函数)，结果表现为上图的反射光的叶状区域。

$G$描述了微表面的可见性。

注意该等式用于微观层面沿半球积分：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-25-07-image.png)

## 非导体和导体

入射光与表面接触后反射为两个部分：漫反射和高光反射：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-29-08-image.png)

微观层面，部分如入射光会进入表面，在内部散射，然后作为漫反射部分离开表面：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-31-16-image.png)

对于理论上的纯金属材质，并不会发生次表面散射，也就是没有漫反射部分。非导体会散射光，也就是同时拥有漫反射和高光反射部分。

为了更好了构建BRDF，需要区分非导体和导体：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-34-36-image.png)

## 能量守恒

即反射光能量小于入射光能量。

## 高光BRDF

对于高光项，BRDF如下：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-38-45-image.png)

### 法线分布函数

GGX:

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-39-59-image.png)

GLSL实现：

```glsl
float D_GGX(float NoH, float roughness) {
    float a = NoH * roughness;
    float k = roughness / (1.0 - NoH * NoH + a * a);
    return k * k * (1.0 / PI);
}
```

可以使用half float来改进。使用half float时，$1-(n\cdot h)^2$的计算会存在问题，其一，$(n\cdot h)^2$接近1时浮点部分丢失，其二，$n\cdot h$在1附近没有足够的精度。

引入拉格朗日等式：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-46-25-image.png)

优化实现：

```glsl

#define MEDIUMP_FLT_MAX    65504.0
#define saturateMediump(x) min(x, MEDIUMP_FLT_MAX)

float D_GGX(float roughness, float NoH, const vec3 n, const vec3 h) {
    vec3 NxH = cross(n, h);
    float a = NoH * roughness;
    float k = roughness / (dot(NxH, NxH) + a * a);
    float d = k * k * (1.0 / PI);
    return saturateMediump(d);
}
```

### 几何遮蔽

V_Smith-GGX等式：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-50-46-image.png)

GLSL实现：

```glsl
float V_SmithGGXCorrelated(float NoV, float NoL, float roughness) {
    float a2 = roughness * roughness;
    float GGXV = NoL * sqrt(NoV * NoV * (1.0 - a2) + a2);
    float GGXL = NoV * sqrt(NoL * NoL * (1.0 - a2) + a2);
    return 0.5 / (GGXV + GGXL);
}
```

分母中根号下的项范围在[0,1]，可以如下优化：

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-53-03-image.png)

GLSL实现：

```glsl
float V_SmithGGXCorrelatedFast(float NoV, float NoL, float roughness) {
    float a = roughness;
    float GGXV = NoL * (NoV * (1.0 - a) + a);
    float GGXL = NoV * (NoL * (1.0 - a) + a);
    return 0.5 / (GGXV + GGXL);
}
```

### 菲涅尔

Schlick:

![](C:\Users\WIN10\AppData\Roaming\marktext\images\2022-09-06-21-55-00-image.png)

GLSL实现：

```glsl
vec3 F_Schlick(float u, vec3 f0, float f90) {
    return f0 + (vec3(f90) - f0) * pow(1.0 - u, 5.0);
}
```

$f_{90}$可以近似为1：

```glsl
vec3 F_Schlick(float u, vec3 f0) {
    float f = pow(1.0 - u, 5.0);
    return f + f0 * (1.0 - f);
}
```








