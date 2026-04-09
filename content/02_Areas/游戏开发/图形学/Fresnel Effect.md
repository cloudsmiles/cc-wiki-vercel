原理：Fresnel Effect，菲涅耳效应，根据观察角度产生不同反射率从而对表面效果产生影响，当你靠近时，会反射更多的光。菲涅耳效应节点通过计算表面法线与视线方向的夹角来近似。这个角度越大，返回值越大。这种效果经常被用来实现边缘照明，这在很多艺术风格中都很常见

## 边缘光
通过物体表面法向量与view Direction的点积结果，决定是否在边缘，点积结果越小，越靠近边缘

![[Pasted image 20241217010024.png]]


## 带有方向的边缘光

增加一个normal法向量与方向向量的点积结果，再通过step改变平滑效果，与边缘光相乘

通过方向向量可以改变【阴影部分】，最终会影响边缘光

![[Pasted image 20241217235042.png]]


## 裁剪
打开apha clip以及设置alpha的值

位置的Y分量，参与Step计算，调整临界值，使得白色比例变化，最后影响alphaTest
![[Pasted image 20241218022450.png]]


## 裁剪附带边缘光

首先利用Smoothstep做出一个边缘渐变

Smoothstep：如果输入In的值分别在输入Edge1和Edge2的值之间，则返回0和1之间的平滑Hermite插值的结果。如果输入In的值小于输入Step1的值，则返回0；如果大于输入Step2的值，则返回1

最终结果就是画出一条横线，横线的位置随着临界值变更而变更，横线的大小与临界值之差相关

![[Pasted image 20241218233940.png]]


结合裁剪的部分，通过位置的Y分量与临界值，调整显示的比例
![[Pasted image 20241218023700.png]]


## Unity Shader代码
### built in

```shell
Shader "CustomRenderTexture/RimLight"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" { }
        _RimColor ("Rim Color", color) = (1.0, 0, 0, 1.0)
        _RimPower ("Rim Power", Range(0.0001, 3.0)) = 1.0
        _RimIntensity ("Rim Intensity", Range(0, 100.0)) = 1.0
    }
    SubShader
    {
        Tags { "RenderType" = "Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"
            #include "Lighting.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
                float3 worldNormal : TEXCOORD1;
                float3 worldViewDir : TEXCOORD2;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _RimColor;
            half _RimPower;
            float _RimIntensity;

            v2f vert(appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldViewDir = WorldSpaceViewDir(v.vertex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldViewDir = normalize(i.worldViewDir);

                fixed4 diffuse = tex2D(_MainTex, i.uv);
                // pow起衰减作用
                fixed4 rimLight = _RimColor * pow(saturate(1.0 - dot(worldNormal, worldViewDir)), 1.0 / _RimPower) * _RimIntensity;
                //fixed4 rimLight = _RimColor * saturate(1.0 - dot(worldNormal, worldViewDir)) * _RimIntensity;

                fixed3 color = diffuse.rgb + rimLight;
                return fixed4(color, 1.0);
            }
            ENDCG
        }
    }
}


```

### urp
```shell
Shader "CustomRenderTexture/RimLight"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" { }
        _RimColor ("Rim Color", color) = (1.0, 0, 0, 1.0)
        _RimPower ("Rim Power", Range(0.0001, 3.0)) = 1.0
        _RimIntensity ("Rim Intensity", Range(0, 100.0)) = 1.0
    }
    SubShader
    {
        Pass
        {
            Tags { "RenderPipeline" = "UniversalPipeline" }
            Tags { "LightMode" = "UniversalForward" }
            
            HLSLPROGRAM

            #pragma vertex vert
            #pragma fragment frag
            
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
                float3 worldNormal : TEXCOORD1;
                float3 worldViewDir : TEXCOORD2;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;
            half4 _RimColor;
            half _RimPower;
            float _RimIntensity;

            v2f vert(appdata v)
            {
                v2f o;
                o.vertex = TransformObjectToHClip(v.vertex.xyz);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                o.worldNormal = TransformObjectToWorldNormal(v.normal);
                o.worldViewDir = GetWorldSpaceViewDir(v.vertex.xyz);
                return o;
            }

            half4 frag(v2f v) : SV_Target
            {
                // 边缘光算法，求法向量与视线方向的点积，点积越接近0，越靠近边缘
                half3 worldNormal = normalize(v.worldNormal);
                half3 worldViewDir = normalize(v.worldViewDir);

                half4 diffuse = tex2D(_MainTex, v.uv);

                // 生成边缘光颜色，pow起衰减作用，intensity起加强作用
                half4 rimLight = _RimColor * pow(saturate(1.0 - dot(worldNormal, worldViewDir)), 1.0 / _RimPower) * _RimIntensity;

                // 叠加颜色
                half3 color = diffuse.rgb + rimLight.rgb;
                return half4(color, 1.0);
            }
            ENDHLSL
        }
    }
}

```