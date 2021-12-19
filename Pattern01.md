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
output.posCS = 1.0 / 0.0; //NaNを入れることでクリップ
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
