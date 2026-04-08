# 基本概念
- GPU Instancing允许在一次渲染调用中绘制同一网格(Mesh)的多个副本

- 每个实例可以有不同的位置、旋转、缩放等属性

- 这些属性数据会被一次性发送到GPU

# 工作原理
- 将相同模型的多个实例的变换矩阵打包成数组

- 在一次Draw Call中将这些数据发送给GPU

- GPU为每个实例并行处理渲染

# 主要优势

- 性能提升

- 减少CPU到GPU的通信开销

- 显著减少Draw Call数量

- 提高渲染效率

2. 内存优化

- 共享相同的网格和材质数据

- 只需存储实例的变换数据

# 示例

```c#
public class InstancedRenderer : MonoBehaviour 
{
    public Mesh mesh;
    public Material material;
    public int instanceCount = 100;
    
    private Matrix4x4[] matrices;
    
    void Start()
    {
        // 初始化矩阵数组
        matrices = new Matrix4x4[instanceCount];
        
        // 设置每个实例的变换
        for(int i = 0; i < instanceCount; i++) 
        {
            Vector3 position = Random.insideUnitSphere * 10;
            matrices[i] = Matrix4x4.TRS(
                position,
                Quaternion.identity,
                Vector3.one
            );
        }
    }
    
    void Update()
    {
        // 使用GPU Instancing渲染
        Graphics.DrawMeshInstanced(mesh, 0, material, matrices);
    }
}
```


## MaterialPropertyBlock
MaterialPropertyBlock是一种可以为单个渲染实例设置材质属性的方式，而无需创建新的材质实例。
```c#
public class MaterialPropertyBlockExample : MonoBehaviour 
{
    private MaterialPropertyBlock propBlock;
    private Renderer renderer;
    
    void Start()
    {
        // 初始化
        propBlock = new MaterialPropertyBlock();
        renderer = GetComponent<Renderer>();
        
        // 设置颜色
        propBlock.SetColor("_Color", Color.red);
        
        // 应用属性块
        renderer.SetPropertyBlock(propBlock);
    }
    
    // 动态更新颜色示例
    void UpdateColor(Color newColor)
    {
        // 获取当前属性
        renderer.GetPropertyBlock(propBlock);
        
        // 修改颜色
        propBlock.SetColor("_Color", newColor);
        
        // 重新应用
        renderer.SetPropertyBlock(propBlock);
    }
}
```


### 主要用途

- 动态修改材质属性

- 节省内存

- 避免材质实例化

- 配合GPU Instancing使用

### 工作原理

- 创建一个属性块

- 设置需要的属性值

- 将属性块应用到渲染器上

- 这些修改不会影响使用相同材质的其他对象


# 参考
- https://blog.csdn.net/a71468293a/article/details/144633805
- 