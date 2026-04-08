### 主要区别

1. 变体生成时机
```
// multi_compile：始终生成所有变体

#pragma multi_compile _ _MAIN_LIGHT_SHADOWS

  

// shader_feature：仅生成使用过的变体

#pragma shader_feature _NORMALMAP
```

2. 内存使用
```
// multi_compile：所有变体都会被打包

#pragma multi_compile _FOG_OFF _FOG_LINEAR _FOG_EXP _FOG_EXP2

// 生成4个变体，都会被打包

  

// shader_feature：只打包用过的变体

#pragma shader_feature _EMISSION

// 只打包使用过的变体
```

3. 应用场景
```
// multi_compile：适用于运行时经常切换的特性

#pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE

  

// shader_feature：适用于材质级别的静态特性

#pragma shader_feature _METALLICGLOSSMAP
```

### 具体示例

1. 多光源支持
```
// 使用 multi_compile，因为光源数量可能运行时变化

#pragma multi_compile _ _ADDITIONAL_LIGHTS

#pragma multi_compile _ _ADDITIONAL_LIGHT_SHADOWS

  

void frag(...)

{

#ifdef _ADDITIONAL_LIGHTS

// 处理额外的光源

#endif

}
```

2. 法线贴图
```
// 使用 shader_feature，因为是材质级别的静态设置

#pragma shader_feature _NORMALMAP

  

Properties

{

[NoScaleOffset] _NormalMap("Normal Map", 2D) = "bump" {}

}

  

void frag(...)

{

#ifdef _NORMALMAP

// 使用法线贴图

#else

// 使用默认法线

#endif

}
```

