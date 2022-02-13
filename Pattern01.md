//// Code Pattern集01 /////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# 自前Blit
三角形で描画(quadで描くより 無駄スレッドがでない)  
C#側
```
_CmdBuffer.SetRenderTarget(BuiltinRenderTextureType.CameraTarget,
	RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store,     // color
	RenderBufferLoadAction.DontCare, RenderBufferStoreAction.DontCare); // depth

_CmdBuffer.DrawProcedural(Matrix4x4.identity, m_CopyBackToCamMaterial, 0, MeshTopology.Triangles, 3, 1, null);
```
Shader側
```
output.posCS   = GetFullScreenTriangleVertexPosition(input.vertexID);
output.xyAndUv = GetFullScreenTriangleTexCoord(input.vertexID).xyxy *float4(_ScreenParams.xy, 1.0, 1.0);
```


---
# DepthのMSAAリゾルブ
サンプルの集計にminをとる、maxを取る、どちらも一長一短。  
HDRPでは、UNITY_REVERSED_Zならmin、そうでなければmax。  
  
でも草みたいな細いもののcontactShadowを このデプスから算出する場合、逆転した方が安定したりもする。  
  
自前CopyDepth.shaderでは贅沢して、1ピクセル内のmin, max集計のうち、平均値に近い方、つまり雑な多数決で選んでみた。
```
UNITY_UNROLL
for (int i = 0; i < MSAA_SAMPLES; ++i)
{
	float theDepth = LOAD_TEXTURE2D_MSAA(_CameraMsaaAttachment, coord, i).x;
	outDepthMin = min(theDepth, outDepthMin);
	outDepthMax = max(theDepth, outDepthMax);
	outDepthSum += theDepth;
}

float outDepthAve = outDepthSum * INV_MSAA_SAMPLES;
float2 outDepthDiff = float2(outDepthMax, outDepthAve) - float2(outDepthAve, outDepthMin);
float outDepth = (outDepthDiff.x > outDepthDiff.y)? outDepthMin : outDepthMax;
```


---
# 頂点シェーダ内でクリップする
```
#pragma warning (disable : 4008) // I div by zero for Nan, to clip vertex!

...
output.posCS = 0.0 / 0.0; //NaNを入れることでクリップ
```


---
# unity_InstanceIDを元に インスタンスをタイル状にワールド配置する。
通常(SRPをワールド絶対座標系でやっている場合)
```
#pragma instancing_options procedural:SetupPrcdInstance

void SetupPrcdInstance()
{
#ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
	float4 translateWS;
	translateWS.xyz = CB_CAMERA_TARGET_WS;
	translateWS.w   = 1.0;
	translateWS.x  += (unity_InstanceID % TILE_REPEAT     - TILE_REPEAT * 0.45);
	translateWS.z  += (floor(unity_InstanceID * INV_TILE_REPEAT) - TILE_REPEAT * 0.45);
	UNITY_MATRIX_M._14_24_34_44 = translateWS;
#endif
}
```

SRPがワールド相対座標系の場合、UNITY_MATRIX_Mは関数なので値を上書きできない。  
よって、vertexShader内で直接 加工する必要がある
```
	float4 proceduralTranslateWS = 0.0;
#ifdef UNITY_PROCEDURAL_INSTANCING_ENABLED
	proceduralTranslateWS.xz  = floor( (CB_TV_CENTER_WS.xz + CB_CAMERA_POS_AWS.xz) / TILE_SIZE +0.5) *TILE_SIZE;
	proceduralTranslateWS.x  += (unity_InstanceID % TILE_REPEAT - TILE_REPEAT * 0.45) *TILE_SIZE;
	proceduralTranslateWS.z  += (floor(unity_InstanceID * INV_TILE_REPEAT) - TILE_REPEAT * 0.45) *TILE_SIZE;

    proceduralTranslateWS.xz -= CB_CAMERA_POS_AWS.xz;
    proceduralTranslateWS.y   = CB_TV_CENTER_WS.y;
#endif

    float3 pivotPosOS = float3(-input.uv1.r, 0.0, -input.uv1.g);
	float4 pivotPosWS = float4(pivotPosOS, 1.0) + proceduralTranslateWS; 
```


---
# Box Projection Cube map
```
//. 反射ベクトルをAABBボックス変換
half3 reflWS = reflect( viewDirWS, normal);

float blendDistance = unity_SpecCube1_ProbePosition.w;
float4 cubeCenterWS = unity_SpecCube0_ProbePosition;
float3 cubeMinWS    = unity_SpecCube0_BoxMin - float4( blendDistance,blendDistance,blendDistance,0.0);
float3 cubeMaxWS    = unity_SpecCube0_BoxMax + float4( blendDistance,blendDistance,blendDistance,0.0);

UNITY_BRANCH if (cubeCenterWS.w > 0.0)
{
	float3 nrDir = normalize(reflWS);

	float3 rbMax = (cubeMaxWS - posWS) / nrDir;
	float3 rbMin = (cubeMinWS - posWS) / nrDir;
	float3 rbMinMax = (nrDir > 0.0f) ? rbMax : rbMin;

	float fa = min(min(rbMinMax.x, rbMinMax.y), rbMinMax.z);

	posWS -= cubemapCenterWS.xyz;
	reflWS = posWS + nrDir * fa;
}

//. 環境マップフェッチ
half4 env = SAMPLE_TEXTURECUBE_LOD( unity_SpecCube0, samplerunity_SpecCube0, reflWS, mipRoughness)

//. HDR値の復元
real alpha = max(unity_SpecCube0_HDR.w * (env.a - 1.0) + 1.0, 0.0);
return (unity_SpecCube0_HDR.x * PositivePow(alpha, unity_SpecCube0_HDR.y)) * env.rgb;
```

---
# 法線マップの合成
Tangent空間法線マップの合成には [いくつかの流派](https://blog.selfshadow.com/publications/blending-in-detail/)がある。ガチ寄りといえばTBN空間での回転を行うものであった。  
とりあえず、PhotoshopでOverLay合成するのは、やめた方が良い。ぱっと見 合成できているだけで、値を壊しているので。  
Unity（の主にHDRP）では[SurfaceGradientを利用した合成](https://blog.unity.com/ja/technology/normal-map-compositing-using-the-surface-gradient-framework-in-shader-graph)が導入されており、イケている。([論文](https://jcgt.org/published/0009/03/04/)) コードとしては[SRP.core.SurfaceGradient.hlsl](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.core/ShaderLibrary/NormalSurfaceGradient.hlsl)が利用できる、これはTriplanerマッピングなど各種方式をSurfaceGradで扱う 統合的なライブラリになっている。  
基本だけ扱うなら以下のような感じ
```
float2 GetDerivTSFromNormTS(float3 normTS)
{
    float2 rcpz = rcp(max(normTS.z, FLT_EPS));
    return -rcpz * normTS.xy;
}

float3 GetSurfGradWSFromDerivTS(float2 derivTS, float3 vTangentWS, float3 vBinormalWS)
{
    return derivTS.x * vTangentWS + derivTS.y * vBinormalWS;
}

float3 GetNormalWSFromSurfGradWS(float3 surfGradWS, float3 vNormalWS)
{
    return SafeNormalize(vNormalWS - surfGradWS);
}
```
元の法線からの傾き量を 非リニアな2次元空間（傾きが強いとビヨーンって長くなるイメージ）で現すことで、それらを線形補完で合成可能にしている。  
同じTBN空間上の値であればDerivativeTS同士で合成可能だし、SurfaceGradientはWorldSpace値なので好きに合成可能。  
実用してみての注意点としては、ハードウェアによってはFLT_EPSだとキツすぎて計算エラーするので、もう少し保守的な値でクリップするのが良い。  
また、SurfaceGrad合成系で、mikkt空間でベイクした法線マップを運用するケースは[Matrix.md](https://github.com/ekakiya/shaderFragment/blob/master/Matrix.md)を参照。
