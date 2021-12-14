//// 座標空間関連 //////////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# 座標空間の種類
[SRP.core/Common.hlsl参照](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl)  
```
AWS: ワールド絶対座標      Absolute world space
WS : カメラ相対座標       World space
VS : ビュー空間座標       View space
OS : オブジェクト空間座標  Object space
CS : クリップ空間座標      Homogenous clip spaces
TS : タンジェント空間座標  Tangent space
SCS: ピクセル座標         Screen space (2Dスクリーン座標, ピクセル単位)
NDC: デバイス座標         Homogeneous normalized device coordinates (2Dスクリーン座標, 正規単位)

SWS: シャドウマップ空間座標	 Shadowmap world space
FWS: 畳んだワールド絶対座標	 Folded world space (AWSを1024で畳んだもの, 広域データで ワールドテクスチャ投影にAWSが大きな値すぎるケースなどに利用)

//. 使ってない
TXS: UVtexture space
RWS: HDRPにおける、カメラ相対座標。HDRPでは WSは絶対座標のままで扱われている。  Camera-Relative world space
```


---
# 座標変換
## 頂点シェーダー内で いつものやつ
```
posWS = mul(UNITY_MATRIX_M,  float4(input.posOS.xyz, 1.0));
posCS = mul(UNITY_MATRIX_VP, posWS);
```

## タンジェント空間 基本
```
normalWS   = normalize( mul( (float3x3) UNITY_MATRIX_M, normalOS)); //ASSUME_UNIFORM_SCALING
tangentWS  = float4( normalize( mul( (float3x3) UNITY_MATRIX_M, tangentOS.xyz)), tangentOS.w);
sign       = tangentWS.w * UNITY_SCALE_SIGN;
binormalWS = cross( normalWS, tangentWS.xyz) * sign;
```

## タンジェント空間 mikkt考慮, 法線マップのsurface gradient合成考慮
VertexShader側
```
float3 normalWS   = SafeNormalize( mul( (float3x3) UNITY_MATRIX_M, normalOS)); //ASSUME_UNIFORM_SCALING
float4 tangentWS  = float4( SafeNormalize( mul( (float3x3) UNITY_MATRIX_M, tangentOS.xyz)), tangentOS.w);
```

FragmentShader側
```
float3 unnormalizedNormalWS = input.normalWS;
float renormFactor = 1.0 * rcp(max(FLT_MIN, length(unnormalizedNormalWS)));	
float sign = (input.tangentWS.w > 0.0 ? 1.0 : -1.0) * UNITY_SCALE_SIGN;

float3 binormalWS = cross(unnormalizedNormalWS, input.tangentWS.xyz) * sign * renormFactor;
float3 tangentWS  = input.tangentWS.xyz * renormFactor;
float3 normalWS   = unnormalizedNormalWS * renormFactor;
```

Frgment内 法線マップ適用時（その後さらにタンジェント空間が必要な場合）
```
float2 normalMapTS = packedNormal.wy *2.0 - 1.0;
float2 derivTS = GetDerivTSFromNormTS(normalMapTS);
float3 surfGradWS = GetSurfGradWSFromDerivTS(derivTS, tangentWS, binormalWS);

float3 normal2WS   = normalize(normalWS - surfGradWS);
float3 tangent2WS  = normalize(tangentWS - dot(tangentWS, normal2WS) * normal2WS);
float3 newBB       = cross(normal2WS, tangent2WS);
float3 binormal2WS = newBB * FastSign(dot(newBB, binormalWS));
```

## タンジェントスペース 視線ベクトル
VertexShader内
```
float3   binormalOS      = cross( normalize( normalOS), normalize( tangentOS.xyz)) * tangentOS.w;
float3x3 objectToTangent = float3x3( tangentOS.xyz, binormalOS, normalOS);
float3   camPosOS        = mul( UNITY_MATRIX_I_M, float4(0, 0, 0, 1)).xyz; //カメラ相対座標でなければ0,0,0のかわりにCB_CAMERA_POS_WS;
float3   viewDirOS       = camPosOS - posOS.xyz;
float3   viewDirTS       = mul( objectToTangent, viewDirOS);
```

## ポスプロComputeShader内、Deferred的な用途でWSを扱う
```
 uint2 posSCS  = Gid * TILE_SIZE + DecodeMorton2D(GTid); //Computeスレッドを画面のピクセルに割り当て
 float2 posNDC = posSCS * PPS_SCREEN_ZW; //PPS_SCREEN_ZWは画面解像度XYの逆数

 float depth = LOAD_TEXTURE2D(_DepthTex, posSCS).r;

 float4 posCS = float4(posNDC * 2.0 - 1.0, depth, 1.0);
#if UNITY_UV_STARTS_AT_TOP
 posCS.y = -posCS.y;
#endif

 float4 posWS = mul(UNITY_MATRIX_I_VP, posCS);
 posWS.xyz /= posWS.w;
```

## その他 空間変換メモ
[SRP.core/SpaceTransform.hlsl参考](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl)  
```
posVS  = mul(UNITY_MATRIX_V, float4(posWS, 1.0)).xyz;
```

[URP/ShaderVariablesFunctions.hlsl/GetVertexPositionInputs参考](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.universal/ShaderLibrary/ShaderVariablesFunctions.hlsl)  
```
float4 ndc = posCS * 0.5f;
posNDC.xy = float2(ndc.x, ndc.y * _ProjectionParams.x) + ndc.w;
posNDC.zw = posCS.zw;
```


---
# CBuffer Matrix系
## matrix
C#側
```
foldedCameraPosition = Vector3(cameraPosition.x % 1024.0f, cameraPosition.y % 1024.0f, cameraPosition.z % 1024.0f);
Matrix4x4 projMatrix = GL.GetGPUProjectionMatrix(camera.projectionMatrix, true); //renderTexture使ってる>True>上下反転など
Matrix4x4 viewMatrix = camera.worldToCameraMatrix;
viewMatrix.SetColumn(3, new Vector4(0, 0, 0, 1)); //camera relative化
Matrix4x4 viewProjMatrix = projMatrix * viewMatrix;
Matrix4x4 invViewProjMatrix = viewProjMatrix.inverse;

_CmdBuffer.SetGlobalMatrix(ShaderId._ViewMatrix, viewMatrix);
_CmdBuffer.SetGlobalMatrix(ShaderId._InvViewMatrix, viewMatrix.inverse);
_CmdBuffer.SetGlobalMatrix(ShaderId._ProjMatrix, projMatrix);
_CmdBuffer.SetGlobalMatrix(ShaderId._InvProjMatrix, projMatrix.inverse);
_CmdBuffer.SetGlobalMatrix(ShaderId._ViewProjMatrix, viewProjMatrix);
_CmdBuffer.SetGlobalMatrix(ShaderId._InvViewProjMatrix, invViewProjMatrix);
_CmdBuffer.SetGlobalVector(ShaderId._WorldSpaceCameraPos, cameraPosition);
_CmdBuffer.SetGlobalVector(ShaderId._FoldedWorldSpaceCameraPos, foldedCameraPosition);
```

Shader側
```
//. WS=カメラ相対座標でやる。 
///Core.UnityInstancing.hlsl, Core.SpaceTransforms.hlslなどは このフラグを受けて 相対座標に対応する。
#define SHADEROPTIONS_CAMERA_RELATIVE_RENDERING (1)
#define MODIFY_MATRIX_FOR_CAMERA_RELATIVE_RENDERING

#define UNITY_MATRIX_M     ApplyCameraTranslationToMatrix(unity_ObjectToWorld) //必須
#define UNITY_MATRIX_I_M   ApplyCameraTranslationToInverseMatrix(unity_WorldToObject)
#define UNITY_MATRIX_V     _ViewMatrix
#define UNITY_MATRIX_I_V   _InvViewMatrix
#define UNITY_MATRIX_P     OptimizeProjectionMatrix(_ProjMatrix) //vfxGraphで必須 //core/SpaceTransforms.hlsl > GetViewToHClipMatrix
#define UNITY_MATRIX_I_P   _InvProjMatrix
#define UNITY_MATRIX_VP    _ViewProjMatrix //必須
#define UNITY_MATRIX_I_VP  _InvViewProjMatrix //ポスプロでSCSからWSを復元するのに便利

#define UNITY_RAW_MATRIX_M     unity_ObjectToWorld
#define UNITY_RAW_MATRIX_I_M   unity_WorldToObject
#define UNITY_SYS_MATRIX_VP    unity_MatrixVP
#define UNITY_SCALE_SIGN       unity_WorldTransformParams.w

//. built-inや公式SRPで見かけたが 必要にならない
//#define UNITY_MATRIX_MV    mul(UNITY_MATRIX_V, UNITY_MATRIX_M)
//#define UNITY_MATRIX_T_MV  transpose(UNITY_MATRIX_MV)
//#define UNITY_MATRIX_IT_MV transpose(mul(UNITY_MATRIX_I_M, UNITY_MATRIX_I_V))
//#define UNITY_MATRIX_MVP   mul(UNITY_MATRIX_VP, UNITY_MATRIX_M)

// optional
#define UNITY_RAW_MATRIX_M     unity_ObjectToWorld
#define UNITY_RAW_MATRIX_I_M   unity_WorldToObject
#define UNITY_SYS_MATRIX_VP    unity_MatrixVP
#define UNITY_SCALE_SIGN       unity_WorldTransformParams.w

#define CB_PIVOT_POS_AWS		unity_ObjectToWorld._m03_m13_m23
#define CB_PIVOT_POS_WS			PosAWSToPosWS(unity_ObjectToWorld._m03_m13_m23)

//. ここでGPUインスタンス対応。UnityInstancing.hlslをベースに 好きにやる
#include "Instancing.hlsl"

//. matrixからベクトルを抽出
#define CB_M_RIGHT_VECTOR    normalize(UNITY_RAW_MATRIX_M._m00_m10_m20)
#define CB_M_UP_VECTOR       normalize(UNITY_RAW_MATRIX_M._m01_m11_m21)
#define CB_M_FORWARD_VECTOR  normalize(UNITY_RAW_MATRIX_M._m02_m12_m22)
#define CB_V_RIGHT_VECTOR    normalize(UNITY_MATRIX_V._m00_m01_m02)
#define CB_V_UP_VECTOR       normalize(UNITY_MATRIX_V._m10_m11_m12)
#define CB_V_FORWARD_VECTOR -normalize(UNITY_MATRIX_V._m20_m21_m22)

//. cameraオプション
#define CB_CAMERA_POS_AWS   	_WorldSpaceCameraPos
#define CB_CAMERA_POS_FWS   	_FoldedWorldSpaceCameraPos
#define CB_CAMERA_FLIP          _ProjectionParams.x
#define CB_CAMERA_NEAR_PLANE	_ProjectionParams.y
```

- \_ProjectionParams //set by unity
	+ x = 1 or -1(if projection is flipped)
	+ y = near plane
	+ z = far plane
	+ w = 1/far plane

- \_ScreenParams //set by unity
	+ x = width
	+ y = height
	+ z = 1 + 1.0/width
	+ w = 1 + 1.0/height

- \_ZBufferParams //set by unity
	+ x = 1-far/near
	+ y = far/near
	+ z = x/far
	+ w = y/far

- unity_OrthoParams //set by unity //vfxGraphで必須 //VFXCommon.cginc > IsPerspectiveProjection
	+ x = orthographic camera's width
	+ y = orthographic camera's height
	+ z = unused
	+ w = 1.0 if camera is ortho, 0.0 if perspective

- unity_WorldTransformParams
	+ x = no use
	+ y = no use
	+ z = no use
	+ w = usually 1.0, or -1.0 for odd-negative scale transforms


## shadowmap matrix
ex.並行光源  
C#側
```
shadowRenderSetting = new Vector4(-light.light.shadowBias * k_ShadowTexelSize, -light.light.shadowNormalBias * k_ShadowTexelSize, 0.0f, 0.0f);

if (!_CullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(
		sunLightIndex,
		0,  //今から描くのがカスケードの どの段階か[0～3]
		1,  //シャドーマップを何枚かくか[1～4], 1でカスケードOFF
		Vector3.right,
		k_ShadowMapSize,
		light.light.shadowNearPlane,
		out shadowViewMatrix, out shadowProjectionMatrix, out shadowSplitData
	))
	return false;

//. shadowBias,shadowNormalBias値を、カメラ画角に応じて補正
float frustumSize = 2.0f / _Dat.shadow.shadowProjectionMatrix.m00;
shadowRenderSetting.x *= frustumSize;
shadowRenderSetting.y *= frustumSize;

cameraPositionSWVS = shadowViewMatrix * new Vector4(cameraPosition.x, cameraPosition.y, cameraPosition.z, 0.0f);
shadowViewMatrix.SetColumn(3, new Vector4(shadowViewMatrix.m03 + cameraPositionSWVS.x, shadowViewMatrix.m13 + cameraPositionSWVS.y, shadowViewMatrix.m23 + cameraPositionSWVS.z, 1));	//camera relative化
shadowDeviceProjectionYFlipMatrix = GL.GetGPUProjectionMatrix(shadowProjectionMatrix, true);
shadowViewDeviceProjectionYFlipMatrix = shadowDeviceProjectionYFlipMatrix * hadowViewMatrix;

_CmdBuffer.SetGlobalMatrix(ShaderId._ViewMatrix, shadowViewMatrix);
_CmdBuffer.SetGlobalMatrix(ShaderId._InvViewMatrix, shadowViewMatrix.inverse);
_CmdBuffer.SetGlobalMatrix(ShaderId._ProjMatrix, shadowDeviceProjectionYFlipMatrix);
_CmdBuffer.SetGlobalMatrix(ShaderId._InvProjMatrix, shadowDeviceProjectionYFlipMatrix.inverse);
_CmdBuffer.SetGlobalMatrix(ShaderId._ViewProjMatrix, shadowViewDeviceProjectionYFlipMatrix);
_CmdBuffer.SetGlobalMatrix(ShaderId._InvViewProjMatrix, shadowViewDeviceProjectionYFlipMatrix.inverse);

```

### shadowmap描画時
Shader側
```
float3 normalWS = normalize(mul((float3x3) UNITY_MATRIX_M, input.normalOS)); //ASSUME_UNIFORM_SCALING
float  invNdotL = 1.0 - saturate(dot(CB_SUN_DIRECTION, normalWS));
float  scale    = invNdotL * _ShadowRenderSetting.y;
posWS.xyz = CB_SUN_DIRECTION * _ShadowRenderSetting.xxx + posWS.xyz;
posWS.xyz = normalWS * scale.xxx + posWS.xyz;
```

- \_ShadowRenderSetting
	+ x = shadow bias
	+ y = shadow normal bias
	+ z =  no use
	+ w = no use

### shadowmap取得時
Shader側
```
float4 posSWS = mul( _WorldToShadowMatrix, float4(posWS, 1.0));
posSWS.xyz /= posSWS.w;

real  tentWeights[9];
real2 tentUVs[9];
SampleShadow_ComputeSamples_Tent_5x5( _ShadowSampleSetting.xxyy, posSWS.xy, tentWeights, tentUVs);

float atten = 0.0;
for (int i = 0; i < 9; i++) {
	atten += tentWeights[i] * SAMPLE_TEXTURE2D_SHADOW(_ShadowMap, s_linear_clamp_compare_sampler, float3(tentUVs[i].xy, posSWS.z));
}
```

- \_ShadowSampleSetting //set by SRP
	+ x = 1/shadowMap Size
	+ y = shadowMap Size
	+ z = no use
	+ w = no use


---
# GPUパイプライン上の、Z値単位の変化と取得
Zn = Znearクリッピングプレーン値  
Zf = Zfarクリッピングプレーン値  
```
頂点の座標
　input.posOS //:オブジェクト空間(実寸)
　↓
{頂点シェーダ内}_______________________________________________________________________________________________________________
　output.posCS //:オブジェクト空間(実寸)posOS > ワールド空間(実寸)posWS > ビュー空間(実寸)posVS
　↓            //   > パースペクティブ空間(ただしzで割る前なので、固定画角90度相当)posCS
　↓            //この時点でのZ値は、Zf側にZn分だけ縮めてあるものの0-1リニア値ではある。
　↓            //w成分にビュー空間でのZ値がはいってる。　//クリッピング空間の同次(homogenous) 座標 算出
　↓
{AUTO:ラスタライズ}____________________________________________________________________________________________________________
　Z = posCS.z /pozCS.w //:パースペクティブ空間 > Zバッファ単位
　↓        　↓		   //Zは、Znで0、Zfで1だがリニアカーブではなく、Zn付近に値幅の大半を使用して精度を稼ぐ 逆ガンマ的カーブになる。
　↓        　↓		   //この時、zだけでなくx,yもwで割られ、x,yに関しては ここでパースペクティブ空間への変換が完了する。
　↓        　↓		   //さらにxyそれぞれの画面解像度が掛けられ、ピクセル単位座標となる。 
　↓        　↓		   //この変換は、vertexShaderとfragmentShaderの間のプログラマブル外処理で行われる。(posCS.xyz /= posCS.w)
　↓        　↓
　↓　　　　{フラグメントシェーダ内}_______________________________________________________________________________________________
　↓        customZ = input.posCS.z + _ZOffset //OPTIONAL:Zバッファ単位で ピクセルオフセット加工したい場合
　↓                                           //非リニア空間でオフセットすることになる。ちゃんと実寸値でオフセットするのは	かなり面倒
　↓
　↓        float dist = GetDist(input.posCS.z);
　↓                   = 1.0 / (_ZBufferParams.z * input.posCS.z + _ZBufferParams.w);
　↓                   //GET:原点からの実寸距離 リニア値
　↓
{AUTO:Zバッファ描き込み}_______________________________________________________________________________________________________
　↓
{ディファードシェーディングのライトパス or ポスプロ内}______________________________________________________________________________
　float depthSq = SAMPLE_DEPTH_TEXTURE( _CameraDepthTexture, uv); //:Zバッファの値
　↓                                                                     ↓
　float depth = Linear01Depth( depthSq);                                ↓
  ↓           = 1.0 / (_ZBufferParams.x * depthSq + _ZBufferParams.y);  ↓
  ↓           //GET:原点からZfまでを0-1に正規化した、リニア値                ↓
  ↓                                                                     ↓
  ↓                                               float dist = LinearEyeDepth( depthSq); 
  ↓                                                          = 1.0 / (_ZBufferParams.z * depthSq + _ZBufferParams.w);
  ↓                                                          = 1.0 / ((1-far/near)/far * depthSq + (far/near)/far); 
  ↓                                                          //GET:原点からの実寸距離 リニア値
  ↓
  //ex. BRP-deferredでの座標変換 //SRP系においてはUNITY_MATRIX_I_VPを用意してシンプルに対応する手法が主流。座標変換の項を参照。
　i.ray = i.ray *( _ProjectionParams.z /i.ray.z);
　float4 vpos = float4( i.ray *depth, 1.0);
　↓			  //GET:頂点位置(ビュー空間 実寸)
　↓			  //パースペクティブ空間でのVをビュー空間に投影した長さ(実寸) * 正規Depth
　↓			  //i.rayはBRP-deferredのvert_deferred内で算出。screenQuadのnormalに仕込んであったりもするみたい。
　↓
　wpos = mul( unity_CameraToWorld, vpos).xyz; //GET:頂点位置(ビュー空間>ワールド空間)
　↓	
　viewDir = wpos - _WorldSpaceCameraPos; //GET:原点からの実寸距離 リニア値 , Vベクトル(ワールド空間) 算出
　float wDepth = viewDir.x +viewDir.y +viewDir.z;
　viewDir = -normalize( viewDir);
　wDepth /= -viewDir.x -viewDir.y -viewDir.z;
```

## ラスタライズ時のdepth変換への理解を深めようメモ
[重要参考](http://marupeke296.com/DXG_No70_perspective.html)  
d = 距離(実寸)  
z = 距離(Zバッファ単位)  
```
d2z:  z = ( d -Zn) *Zf /( d *( Zf -Zn))  // (d:[Zn - Zf]>[0 - 1])/d みたいな感じ。
                                         // Znぶん縮めて Zf距離で正規化したリニア値を、Zfで割ることで、
                                         // Zn付近に値幅の大半を使用して 近景のZ精度を稼ぐ、逆ガンマ的カーブが作れる。

z2d:  d = Zn *Zf /( Zf + z *( Zn -Zf))   // d2zの逆変換。

// z2dは GPUでv2fのあたりで自動的に変換される。

// d2zは LinearEyeDepthで変換できる。この変換を効率的に処理する 為に、Unityでは 
// _ZBufferParams( 1- Zf /Zn , Zf /Zn , ( 1 - Zf /Zn) /Zf , ( Zf /Zn) /Zf)が定義されている。

// _ProjectionParams(座標反転フラグ , Zn , Zf , 1 /Zf)
```
.  
```
z = (d - Zn) /d  *  Zf / (Zf - Zn)
// つまり、カメラからの距離とニアからの距離の割合、ともいえる。
// この割合が、nearで０，farで1になるよう、後半の定数でスケールしている。
```


---
# C#上で MatrixからPosition, Rotation, Scaleの復元をしたい（不具合あり）
```
Vector4 vX = new Vector4(1, 0, 0, 0);
Vector4 vY = new Vector4(0, 1, 0, 0);
Vector4 vZ = new Vector4(0, 0, 1, 0);

private static Matrix4x4 GetQPS(Matrix4x4 m, Vector4 vX, Vector4 vY, Vector4 vZ)
{
	Matrix4x4 QPS = new Matrix4x4();  //[Qx,Qy,Qz,Px], [Qw,0,0,Py], [Sx,Sy,Sz,Pz]
	QPS.m11 = 0.0f;
	QPS.m12 = 0.0f;             //No Use

	QPS.m03 = m.m03;
	QPS.m13 = m.m13;
	QPS.m23 = m.m23;            //Position

	Vector4 vTmpX;
	Vector4 vTmpY;
	Vector4 vTmpZ;

	vTmpX = m * vX;
	vTmpY = m * vY;
	vTmpZ = m * vZ;
 
	QPS.m20 = vTmpX.magnitude;
	QPS.m21 = vTmpY.magnitude;
	QPS.m22 = vTmpZ.magnitude;  //Scale

	Matrix4x4 R;
	R.m00 = m.m00 / QPS.m20;
	R.m01 = m.m01 / QPS.m20;
	R.m02 = m.m02 / QPS.m20;
	R.m10 = m.m10 / QPS.m21;
	R.m11 = m.m11 / QPS.m21;
	R.m12 = m.m12 / QPS.m21;
	R.m20 = m.m20 / QPS.m22;
	R.m21 = m.m21 / QPS.m22;
	R.m22 = m.m22 / QPS.m22;

	float[] elem = new float[4];
	elem[0] =  R.m00 - R.m11 - R.m22 + 1.0f;
	elem[1] = -R.m00 + R.m11 - R.m22 + 1.0f;
	elem[2] = -R.m00 - R.m11 + R.m22 + 1.0f;
	elem[3] =  R.m00 + R.m11 + R.m22 + 1.0f;

	int biggestIdx = 0;
	for (int i = 0; i < elem.Length; i++)
	{
		if (elem[i] > elem[biggestIdx])
		{
			biggestIdx = i;
		}
	}

	if (elem[biggestIdx] < 0)
	{
		Debug.Log("Wrong matrix.");
		return new Matrix4x4();
	}

	float[] q = new float[4];
	float v = Mathf.Sqrt(elem[biggestIdx]) * 0.5f;
	q[biggestIdx] = v;
	float mult = 0.25f / v;

	switch (biggestIdx)
	{
		case 0: //x                         //列優先(OpenGL系)のメモリ配置
			q[1] = (R.m10 + R.m01) * mult;
			q[2] = (R.m02 + R.m20) * mult;
			q[3] = (R.m21 - R.m12) * mult;
			break;
		case 1: //y
			q[0] = (R.m10 + R.m01) * mult;
			q[2] = (R.m21 + R.m12) * mult;
			q[3] = (R.m02 - R.m20) * mult;
			break;
		case 2: //z
			q[0] = (R.m02 + R.m20) * mult;
			q[1] = (R.m21 + R.m12) * mult;
			q[3] = (R.m10 - R.m01) * mult;
			break;
		case 3: //w
			q[0] = (R.m21 - R.m12) * mult;
			q[1] = (R.m02 - R.m20) * mult;
			q[2] = (R.m10 - R.m01) * mult;
			break;
	}

	QPS.m00 = q[0];
	QPS.m01 = q[1];
	QPS.m02 = q[1];
	QPS.m10 = q[3];             //Rotation Quaternion

	return QPS;
}
```