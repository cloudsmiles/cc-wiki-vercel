# 基本概念

SRP Batcher是Unity为可编程渲染管线(URP/HDRP)提供的一种优化Draw Call的技术，主要用于提高材质属性不同但使用相同shader变体的物体的渲染效率。

# 核心原理
1. 传统Draw Call过程
```
每个Draw Call需要：
1. CPU设置材质属性
2. CPU设置GameObject的Transform
3. CPU验证状态
4. CPU提交Draw Call给GPU
5. GPU读取数据并渲染
```

2. 传统批处理的问题
```
动态批处理：
- 合并网格，CPU开销大
- 限制多（顶点数、材质必须相同等）

静态批处理：
- 内存占用大
- 烘焙后不能移动
```


3. SRP Batcher的创新
```
核心思路：
1. 将所有材质属性存入大块连续内存(Chunk)
2. 将所有对象的Transform存入大块连续内存
3. GPU直接从这些内存块读取数据
```

## 工作原理

 1. 数据组织
```c#
// 材质属性内存块示意
struct MaterialProperties {
    float4 _BaseColor;
    float4 _SpecColor;
    // ... 其他属性
};

// Transform数据内存块示意
struct PerObjectData {
    float4x4 unity_ObjectToWorld;
    float4x4 unity_WorldToObject;
    // ... 其他变换数据
};

// 连续内存布局
|--MaterialProperties--|--MaterialProperties--|...  // 材质数据块
|--PerObjectData--|--PerObjectData--|...           // 变换数据块
```

2. 数据访问
```
// Shader中的数据访问
CBUFFER_START(UnityPerMaterial)    // 材质属性块
    float4 _BaseColor;
    float4 _SpecColor;
CBUFFER_END

CBUFFER_START(UnityPerDraw)        // 变换数据块
    float4x4 unity_ObjectToWorld;
    float4x4 unity_WorldToObject;
CBUFFER_END
```


3. 批处理条件
- shader要求
```
// 必须使用CBUFFER包装材质属性
CBUFFER_START(UnityPerMaterial)
    // 所有材质属性必须在这里声明
CBUFFER_END

// 不能在Shader中使用以下方式
float4 _Color;  // ❌ 未使用CBUFFER
```

- 相同的Shader变体

- 相同的Pass

- 相同的关键字


4. 流程对比
传统渲染流程
```
每个对象：
CPU → 设置材质属性 → 设置Transform → 验证状态 → 提交Draw Call → GPU渲染
```

SRP Batcher渲染流程
```
预处理：
CPU → 构建材质属性内存块
CPU → 构建Transform内存块

渲染时：
CPU → 提交内存块引用 → GPU直接读取连续内存 → 渲染
```

# 最佳实践

1. 材质管理

- 使用共享材质

- 最小化材质变体

- 合理组织材质属性


2. Shader编写

- 确保Shader兼容SRP Batcher

- 使用CBUFFER包装属性

- 优化Shader变体

