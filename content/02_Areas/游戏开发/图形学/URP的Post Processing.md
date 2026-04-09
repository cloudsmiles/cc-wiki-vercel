# 需要的组件
- Volume Component - 用于控制效果参数
- Render Feature - 负责将后处理效果注入渲染管线
- Render Pass - 实现具体的渲染逻辑
- Shader - 定义后处理的具体视觉效果

# 代码编写

## 自定义volume组件
创建`CustomPostScreenTint`脚本
`Volume Component`用于在`Inspector`中暴露和控制后处理效果的参数
```c#
using System;
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

[Serializable, VolumeComponentMenu("CustomPostScreenTint")]
public class CustomPostScreenTint : VolumeComponent, IPostProcessComponent
{
    public bool IsActive() => true;

    public bool IsTileCompatible() => true;

    [Tooltip("Intensity of the tint effect")]
    public FloatParameter tintIntensity = new FloatParameter(0.5f);
    [Tooltip("Color to tint the screen with")]
    public ColorParameter tintColor = new ColorParameter(Color.white);

}

```

并将脚本添加到Volume队列  
![将脚本添加到Volume队列](https://i-blog.csdnimg.cn/direct/1d5ce177902944b3b470412df34761a2.png)

## 编写shader
创建`URPScreenTintShader.shader`
后处理`Shader`定义具体的视觉效果

```c#
Shader "CustomPost/URPScreenTintShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _OverlayColor ("Tint Color", Color) = (1,1,1,1)
        _Intensity ("Intensity", Range(0,1)) = 1
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" }

        Pass
        {
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"


            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float4 vertex : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            sampler2D _MainTex;
            float _Intensity;
            float4 _OverlayColor;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = TransformObjectToHClip(v.vertex);
                o.uv = v.uv;
                return o;
            }

            float4 frag (v2f i) : SV_Target
            {
                // sample the texture
                float4 col = tex2D(_MainTex, i.uv);
                col.rgb = lerp(col.rgb, _OverlayColor.rgb, _Intensity);

                // col.rgb *=  _Intensity;

                return col;
            }
            ENDHLSL
        }
    }
}
```


## 创建renderFeature
创建`TintRenderFeature.cs`并编写代码

`Render Feature`是连接`URP`[渲染管线](https://so.csdn.net/so/search?q=%E6%B8%B2%E6%9F%93%E7%AE%A1%E7%BA%BF&spm=1001.2101.3001.7020)的入口点  
`Render Pass`包含具体的渲染逻辑

```c#
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class TintRenderFeature : ScriptableRendererFeature
{

    private TintPass tintPass;

    public override void Create()
    {
        tintPass = new TintPass();
    }
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        renderer.EnqueuePass(tintPass);
    }

    class TintPass : ScriptableRenderPass
    {
        private Material tintMaterial;
        int tintShaderID = Shader.PropertyToID("_Temp");
        RenderTargetIdentifier src, tint;
        public TintPass()
        {
            if (!tintMaterial)
            {
                var shader = Shader.Find("CustomPost/URPScreenTintShader");
                if (shader == null)
                {
                    Debug.LogWarning("Cannot find shader: CustomPost/URPScreenTintShader");
                    return;
                }
                tintMaterial = new Material(shader);
            }
            renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
        }

        public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)
        {

            src = renderingData.cameraData.renderer.cameraColorTarget;
            var descriptor = renderingData.cameraData.cameraTargetDescriptor;
            cmd.GetTemporaryRT(tintShaderID, descriptor);
            tint = new RenderTargetIdentifier(tintShaderID);
        }

        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            CommandBuffer cmd = CommandBufferPool.Get("TintRenderFeature");
            VolumeStack stack = VolumeManager.instance.stack;
            CustomPostScreenTint tintComponent = stack.GetComponent<CustomPostScreenTint>();
            if (tintComponent == null || !tintComponent.IsActive())
            {
                Debug.LogWarning("CustomPostScreenTint is missing or inactive.");
                return;
            }


            tintMaterial.SetColor("_OverlayColor", tintComponent.tintColor.value);
            tintMaterial.SetFloat("_Intensity", tintComponent.tintIntensity.value);

            Blit(cmd, src, tint, tintMaterial);
            Blit(cmd, tint, src);

            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);

        }
        public override void OnCameraCleanup(CommandBuffer cmd)
        {
            cmd.ReleaseTemporaryRT(tintShaderID);
        }
    }
}

```

## 添加renderFeature
在 `URP Asset`中的 `Renderer List` 中添加`RenderFeature`  
![在 URP Asset 中的 Renderer List 中添加RenderFeature](https://i-blog.csdnimg.cn/direct/f1c47ab7143c48b796677473795b63c9.png)

# 参考
- https://blog.csdn.net/a71468293a/article/details/144649558
- https://blog.csdn.net/qq_17347313/article/details/106846631