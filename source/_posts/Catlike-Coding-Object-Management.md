---
title: UnityShader学习
date: 2024-03-28 13:36:25
tags: [unity]
categories: [unity]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/disslove.gif
---
update at 2024.3.28
-------------------------
这个系列只是记录下untiy学习成果，不会有很多的原理性解释。大家随便看看。要学习可以直接看catlike原文
# UnityShader
## Creating a Clock
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/clock.gif)

## Building a Graph

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713031728.png)

## 透明纹理
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230714161935.png)

## animation
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/animation.gif)

## 广告牌
广告牌技术一般应用于简单的多边形模型，通过视角方法来旋转一格被纹理着色的多边形，使得多边形看起来总是面对摄相机。例如：云朵、烟雾、树木等。
广告牌的原理：构建旋转矩阵，基向量是：表面法线（normal），指向上（up）的方向，指向右（right）的方向，还要确定锚点（旋转中心）。
基向量的确定有两种：
- 固定法线（让面片总是朝视角方向）
- 固定$\hat{up}$向量（例如草地）

以固定法线为例，
$\begin{align}
\vec{right} &= \vec{right} \times \vec{normal}\\
\hat{up} &= \hat{normal} \times \hat{right}
\end{align}$

这样就得到了旋转矩阵的正交基。
（固定$\hat{up}$向量，过程类似）
然后再对顶点应用旋转矩阵。
$P_{out}=
\begin{bmatrix}
 & & & 0\\
right& up& normal& 0\\
 & & & 0\\
0 &0 &0 &1
\end{bmatrix}
\begin{bmatrix}
x\\ y\\ z\\ 1
\end{bmatrix}$

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/%E5%B9%BF%E5%91%8A%E7%89%8C.gif)

```glsl
Shader "Unity Shaders Book/Chapter 11/BillBoading" {
	Properties {
		_MainTex ("Main Tex", 2D) = "white" {}
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
	    // 约束垂直方向的程度
		_VerticalBillboarding ("Vertical Restraints", Range(0, 1)) = 1 
	}
    SubShader {
        Tags {
            "Queue" = "Transparent"
            "IgnoreProjector" = "True"
            "RenderType" = "Transparent"
            "DisableBatching" = "True"
        }

        Pass {
            Tags {
                "LightMode" = "ForwardBase"
            }

            Zwrite Off
            Blend SrcAlpha OneMinusSrcAlpha
            Cull Off

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;
            fixed _VerticalBillboarding;

            struct a2v {
                float4 vertex : POSITION;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            v2f vert(a2v v) {
                v2f o;

                // 获取锚点位置
                float3 center = float3(0, 0, 0);
                // 获取模型空间下视角方向
                float3 viewer = mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1));

                // 计算目标法线向量
                float3 normalDir = viewer - center;
                normalDir.y = normalDir.y * _VerticalBillboarding;
                normalDir = normalize(normalDir);

                // 计算（粗略）向上方向
                float3 upDir = abs(normalDir.y) > 0.999 ? float3(0, 0, 1) : float3(0, 1, 0);
                float3 rightDir = normalize(cross(upDir, normalDir));
                // 计算向上方向
                upDir = normalize(cross(normalDir, rightDir));

                // 计算偏移后顶点的模型空间位置
                float3 centerOffs = v.vertex.xyz - center;
                float3 localPos = center + rightDir * centerOffs.x + upDir * centerOffs.y + normalDir * centerOffs.z;

                // 将顶点坐标变换到裁剪空间
                o.pos = UnityObjectToClipPos(float4(localPos, 1));
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);

                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed4 c = tex2D(_MainTex, i.uv);
                c.rgb *= _Color.rgb;

                return c;
            }
            ENDCG
        }
    }
    FallBack "Transparent/VertexLit"
}
```

## 消融效果
消融效果的思路就是对一张噪声图进行采样后透明度测试，再将小于阈值的片元clip掉。而镂空边缘区域的烧焦效果可以通过两种颜色进行混合，再通过pow提高明暗对比，再与原颜色混合得到。

Shader分为两个Pass，第一个Pass按照上述思路实现烧焦效果。第二个Pass实现阴影投射，因为默认的阴影投射是没有镂空的透明度测试的，我们在自定义的ShadowCaster Pass的片元着色器中采用同样的透明度测试来镂空阴影

```glsl
Shader "My Shaders/Dissolve" {
    Properties {
        // 消融程度
        _BurnAmount("Burn Amount", Range(0.0, 1.0)) = 0.0

        // 烧焦效果线宽
        _LineWidth("burn Line Width", Range(0.0, 0.2)) = 0.1

        _MainTex("Base (RGB)", 2D) = "white" {}
        _BumpMap("Normal Map", 2D) = "bump" {}

        // 消融的两种颜色
        _BurnFirstColor("Burn First Color", Color) = (1, 0, 0, 1)
        _BurnSecondColor("Burn Second Color", Color) = (1, 0, 0, 1)

        _BurnMap("Burn Map", 2D) = "white" {}
    }
    SubShader {
        Tags {
            "RenderType" = "Opaque"
            "Queue" = "Geometry"
        }
        // Pass0：实现消融效果
        Pass {
            Tags {
                "LightMode" = "ForwardBase"
            }

            Cull Off

            CGPROGRAM
            #include "Lighting.cginc"
            #include "AutoLight.cginc"

            #pragma vertex vert
            #pragma fragment frag

            #pragma multi_compile_fwdbase

            fixed _BurnAmount;
            fixed _LineWidth;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            sampler2D _BumpMap;
            float4 _BumpMap_ST;
            fixed4 _BurnFirstColor;
            fixed4 _BurnSecondColor;
            sampler2D _BurnMap;
            float4 _BurnMap_ST;


            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float2 uvMainTex : TEXCOORD0;
                float2 uvBumpMap : TEXCOORD1;
                float2 uvBurnMap : TEXCOORD2;
                float3 lightDir : TEXCOORD3;
                float3 worldPos : TEXCOORD4;
                SHADOW_COORDS(5)
            };

            v2f vert(a2v v) {
                v2f o;

                o.pos = UnityObjectToClipPos(v.vertex);
                o.uvMainTex = TRANSFORM_TEX(v.texcoord, _MainTex);
                o.uvBumpMap = TRANSFORM_TEX(v.texcoord, _BumpMap);
                o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);

                // 在切线空间下计算bumpMap
                TANGENT_SPACE_ROTATION;
                o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

                TRANSFER_SHADOW(o);

                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;

                // clip实现消融效果
                clip(burn.r - _BurnAmount);

                // 计算bump
                float3 tangentLlightDir = normalize(i.lightDir);
                fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uvBumpMap));

                fixed3 albedo = tex2D(_MainTex, i.uvMainTex).rgb;

                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLlightDir));

                // 消融边缘过渡
                fixed t = 1 - smoothstep(0.0, _LineWidth, burn.r - _BurnAmount);
                fixed3 burnColor = lerp(_BurnFirstColor, _BurnSecondColor, t);
                burnColor = pow(burnColor, 5); // 增加明暗对比

                UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
                fixed3 finalColor = lerp(ambient + diffuse * atten, burnColor, t);

                return fixed4(finalColor, 1);
            }
            ENDCG
        }
        // Pass1：投射阴影
        Pass{
            Tags{
                "LightMode" = "ShadowCaster"
                }
            
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            #pragma multi_compile_shadowcaster

            #include "UnityCG.cginc"

            fixed _BurnAmount;
            sampler2D _BurnMap;
            float4 _BurnMap_ST;

            struct a2v {
                float4 vertex : POSITION;
                float2 texcoord : TEXCOORD0;
                float3 normal : NORMAL;
            };
            
            struct v2f {
                V2F_SHADOW_CASTER;
                float2 uvBurnMap : TEXCOORD1;
            };

            v2f vert(a2v v) {
                v2f o;

                TRANSFER_SHADOW_CASTER_NORMALOFFSET(o);

                o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);

                return o;
            }

            fixed4 frag(v2f i) : SV_Target{
                fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;

                clip(burn.r - _BurnAmount);

                SHADOW_CASTER_FRAGMENT(i)
            }
            ENDCG
            }
    }
    FallBack "Diffuse"
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/disslove.gif)