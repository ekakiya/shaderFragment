//// Shader命令関連 ///////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# SRP batcher対応ルール
- UnityPerDraw, UnityPerMaterialが仕様に沿って定義されていること。
- TextureやStructuredBufferは これに含まれない。ただしtextureSizeプロパティとかを使う場合 それらは含まれる。
- 以下の 1と2両方を満たしたときに、batcher対応からはずれる。
	+ 1. CBUFFER_START(UnityPerMaterial), CBUFFER_END　で挟んでいないところに定数が定義されていて
	+ 2. その定数がProperties内で同名定義されている
- materialPropertyBlock使った時、batcher対応からはずれる。
- 同一シェーダー内のPass違いでCBUFFERの内容変えたとき、batcher対応からはずれる。

## UnityPerDrawのレイアウト
SRPbatcherの[Blog記事](https://blog.unity.com/ja/technology/srp-batcher-speed-up-your-rendering)公開当時のもの + 2020.2あたりで追加された環境マップ拡張系  
ブロックごとに取捨選択可。一部 floatではなくhalf指定可。  
```
//. Space
///オブジェクト座標系
float4x4 unity_ObjectToWorld;			// Matrix: OS to WS
float4x4 unity_WorldToObject;			// Matrix: WS to OS
float4   unity_LODFade;					// x = fade value ranging within [0,1]	// y = x quantized into 16 levels	// z = no use		// w = no use
float4   unity_WorldTransformParams;	// x = no use								// y = no use						// z = no use		// w = usually 1.0, or -1.0 for odd-negative scale transforms	//h可

//. LigthMap
///ライトマップ用のテクスチャトランスフォーム
float4 unity_LightmapST;		//
float4 unity_DynamicLightmapST;	//

//. Render Layer
///レンダーレイヤー設定、ビットマスク値
float4   unity_RenderingLayer;	// x = asFloat(uint Renderer.renderingLayerMask)

//. SphericalHarmonics
///ライトプローブの値
float4 unity_SHAr;	// xyz = L1の赤成分 // w = L0の赤成分 // (normal.xyz, 1.0)とdotして使う	//h可
float4 unity_SHAg;	// xyz = L1の緑成分 // w = L0の緑成分 //h可
float4 unity_SHAb;	// xyz = L1の青成分 // w = L0の青成分 //h可
float4 unity_SHBr;	// xyzw = L2の赤成分	// (normal.xyzz * normal.yzzx)とdotして使う	//h可
float4 unity_SHBg;	// xyzw = L2の緑成分	//h可
float4 unity_SHBb;	// xyzw = L2の青成分	//h可
float4 unity_SHC;	// xyz = L2の残り成分	// (normal.x * normal.x - normal.y * normal.y)と掛けて使う	//h可

//. ProbeVolume
///ライトプローブProxyVolume(LPPV)の設定
float4   unity_ProbeVolumeParams;			// x = probeVolume enabled? 1 : 0	// y = use local space? 1 : 0	// z = Texel size on U texture coordinate
float4x4 unity_ProbeVolumeWorldToObject;	//
float4   unity_ProbeVolumeSizeInv;			//
float4   unity_ProbeVolumeMin;				//

//. Probe Occlusion
///ShadowMaskを利用している場合、ライトマップをベイクしていないオブジェクトについては、同じ4灯についてシャドウ値がSHから提供され、ここに入る。たしか
float4 unity_ProbeOcclusion;	//

//. Motion Vector
///モーションブラー用設定
float4x4 unity_MatrixPreviousM;
float4x4 unity_MatrixPreviousMI;
float4   unity_MotionVectorsParams;	// x = use last frame positions(as skinMeshRenderer)? 1 : 0	// y = force no motion	// z = Z bias value	// w = Camera only

//. Light Indices
///MeshRendererごとにライト下リングさせてるSRPの場合、ライト番号が提示される
float4 unity_LightData;			//h可
float4 unity_LightIndices[2];	//h可

//. Reflection Probe 0
///環境マップ　基本の1枚 の高輝度復元設定
float4 unity_SpecCube0_HDR;	//h可

//. Reflection Probe 1
///環境マップ　ブレンドする2枚目 の高輝度復元設定
float4 unity_SpecCube1_HDR;	//h可

//. Reflection Probe Box Projection対応
///Unity2020.2くらいで追加。環境マップのブレンドとボックスプロジェクションの設定
float4 unity_SpecCube0_BoxMax;          // xyz = posWS // w = blend distance
float4 unity_SpecCube0_BoxMin;          // xyz = posWS // w = lerp value
float4 unity_SpecCube0_ProbePosition;   // xyz = posWS // w = box projection enabled? 1 : 0
float4 unity_SpecCube1_BoxMax;          // xyz = posWS // w = blend distance
float4 unity_SpecCube1_BoxMin;          // xyz = posWS // w = the sign of (SpecCube0.importance - SpecCube1.importance)
float4 unity_SpecCube1_ProbePosition;   // xyz = posWS // w = box projection enabled? 1 : 0

```


---
# Unity Gpu Instancing
所謂GPUinstancingバッチは、SRP batcherが有効の場合 使用されない。  
ScriptからのGPUinstancing描画はSRP batcherと同時使用可能だが、PerInstancingプロパティはSRP batcherと排他。  
なので、多量に色違いオブジェクトを描くなどのケースでは、使い分けるか、自前で DOTSinstancing的な手法  
(InstanceIDに対応したArrayを用意してComputeBufferにセットし、シェーダで これを読む)を実装する必要がある。  
## マテリアルインスタンス対応
- マテリアルのGPU Instance設定をオン

## 頂点シェーダ対応
- VertexInputに、UNITY_VERTEX_INPUT_INSTANCE_IDを記載			//GPUインスタンス用IDが入る。
- vertex shader冒頭で、UNITY_SETUP_INSTANCE_ID(input);を記載	//UNITY_MATRIX_Mなどが、GPUインスタンス用のCBUFFERからのマトリクスに上書きされる。

## ピクセルシェーダ対応(optional)
- vertex shader冒頭で、UNITY_SETUP_INSTANCE_ID(input);に続いて UNITY_TRANSFER_INSTANCE_ID(input, output);を記載		//fragment shaderにGPUインスタンス用IDを受け渡す。
- VertexOutputに、UNITY_VERTEX_INPUT_INSTANCE_IDを記載																//fragment shaderにGPUインスタンス用IDを受け渡す。
- fragment shader冒頭で、UNITY_SETUP_INSTANCE_ID(input);を記載。													//GPUインスタンス用IDが入る。

## GPU Instance対応プロパティ(optional)
・マテリアルプロパティ宣言時、インスタンス対応を宣言
```
UNITY_INSTANCING_BUFFER_START(PerInstance)
UNITY_DEFINE_INSTANCED_PROP(float4, _Color)
UNITY_INSTANCING_BUFFER_END(PerInstance)
```

・シェーダー内で、値を取得
```
float4 matColor = UNITY_ACCESS_INSTANCED_PROP(PerInstance, _Color);
```


・スプライトの場合は
```
UNITY_INSTANCING_BUFFER_START(PerDrawSprite)
```


---
# SRPのCBuffer
## Render Layer
貴重なUnityPerDrawプロパティ。使っていなければ、インスタンス単位のIDとか仕込むのに使えて便利。  
Cullingには使われず、DrawRenderer時にbitmaskでマスクされる（SRPの設定に拠る）。0にするとさすがに描かれないので注意。  
  
C#側
```
MeshRenderer mr = GetComponent<MeshRenderer>();
mr.renderingLayerMask = theNumber;
```

Shader側
```
uint theNumber = asuint(unity_RenderingLayer.x);
```


---
# Semanticの基本
vertexShaderへの入力
```
struct VertexInput
{
	float4 posOS	 : POSITION;	//頂点位置
	float3 normalOS  : NORMAL;		//法線ベクトル  //optional
	float4 tangentOS : TANGENT;		//タンジェントベクトル  //optional
	float4 uv0       : TEXCOORD0～;	//UVなど汎用  //optional
	float4 vColor    : COLOR;		//頂点カラー  //optional
	uint   vId       : SV_VertexID; //頂点番号  //optional
	UNITY_VERTEX_INPUT_INSTANCE_ID	//GPUインスタンス用ID  //optional
};
```
vertexShaderからfragmentShaderへの出力
```
struct VertexOutput
{
	float4 posCS	 : SV_POSITION;	//頂点位置
	float4 tex       : TEXCOORD0～;	//UVなど汎用 //optional
	UNITY_VERTEX_INPUT_INSTANCE_ID	//GPUインスタンス用ID	//fragment shader内でもID使いたい時用  //optional
};
 //TEXCOORDはtarget4.0で32個まで、target3.0で10個まで、それより古いと8個まで。少ないに越したことはない。
```
fragmentShaderからの出力(基本)
```
half4 frag() : SV_Target
{}
 //color+alphaの出力であればstruct定義は必要とせず、half4やfloat4で出力すれば良い
```
fragmentShaderからの出力(MRTや、カスタムdepth)
```
fvoid rag(
	out half4 outGBuffer0 : SV_Target0,
	out half4 outGBuffer1 : SV_Target1, 
	out half4 outGBuffer2 : SV_Target2, 

	out float outDepth    : SV_Depth //optional
 ){}
	//SV_Depthは単値のZを描きだすので、MSAA中、これ使うとAA効果なくなるっぽいぞ
	//　〃　は宣言したらZ書き出さなきゃいけない縛りなので、cutoutとかするとエラーでるらしいぞ
```


---
# vertexからfragmentに渡すSemanticの補完設定
[参考](https://docs.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-struct)  
```diff
! ex.
nointerpolation float4 tex       : TEXCOORD0;
```

- linear
	+ 無指定だとこれ。普通のリニア補間
- nointerpolation
	+ 補間なし。たぶん、トライアングル0番？頂点の値だけが来てる。
- noperspective
	+ パース補正なし（つまりスクリーンスペース)のリニア補間
- centroid
	+ 三角形の重心補間あり。MSAA必須。トライアングルのエッジ（メッシュのエッジだけではない）でアンチかかる感じがあるが、精度あまいので、グラデがディザったりもするぞ。なので、その後ライン抽出とかやるとlinearより荒れる。
- sample
	+ ShaderModel4.1以降。MSAA必須。pixel中心ではなくsample点で値を取る。これ使った時点で、MSAAの全サンプル点についてfragmentShaderが走る。(SV_SampleIndex読んだ時とかと同じように)。高周波なシェーディングも綺麗ー…なぜならSuperSamplingだから！(順当に負荷が上がる)

## 複数の組み合わせが可能なもの
- centroid noperspective
	+ スクリーンスペース（centroid-adjusted affine）
- centroid linear
- sample noperspective
	+ SSAA強制ポスプロとか
- sample linear


---
# ちょっとマイナーなSemantics
[この辺](https://docs.microsoft.com/en-us/windows/desktop/direct3dhlsl/dx-graphics-hlsl-semantics)にあるものは普通に使える(DirectX固有のものを使った場合、Unityの他API互換サポートは死ぬ)。  
SRPでは、SRP.core/D3D11.hlsl等の中でSV_IsFrontFace系はカバーされている。
  
## VERTEXID_SEMANTIC //頂点番号
```
VertexInput  >> uint vertexID : VERTEXID_SEMANTIC
```
ex.フルスクリーンポスプロを1triangleで描く時に、SRP.coreではGetFullScreenTriangleVertexPosition( vertexID, UNITY_RAW_FAR_CLIP_VALUE );	マクロにより、画面を覆うトライアングル座標やUVが提供されている。

## FRONT_FACE_SEMANTIC //表裏判定
```
VertexOutput >> FRONT_FACE_TYPE cullFace : FRONT_FACE_SEMANTIC
```
シェーダ内で float isFrontVface = IS_FRONT_VFACE(cullFace, 1, 0); など分岐に使う

## SV_Coverage //Alpha to Coverge の出力先
Unityにおいては、shaderのPassにAlphaToMask Onをつけておくことで、AlphaToCoverge機能がオンになる。coverge値はfragmentからの出力アルファ値が使われる。  
さらにps_5_0だと、SV_Coverageをinoutにもできて、ピクセルシェーダ内で値を活用できるらしい  


---
# ddx_fine系
ddx_fine(), ddy_fine()、これらは	ddx(), ddy()の高精度版。  
SHADER_TARGET >= 45かつDirectX固有。Metalだと常にfineかも  
ddxは2x2pixel内でひとつの偏微分値だが、ddx_fineは横2pixelでの偏微分、ddy_fineは縦2pixelでの偏微分になる。精度2倍！（4倍ではない）  


---
# MSAAバッファの、各サンプリング点からのロード
MSAAバッファのサンプリングは UnityのAPIマクロに入っているのだが、テクスチャセットは何故かマクロから抜けている。  
SHADER_TARGET >= 45かつDirectXなら  
```
Texture2DMS<type> textureName  //MSAAテクスチャをsample番号指定Loadできる形でセット
Texture2DMSArray<type> textureName  //MSAAテクスチャアレイをsample番号指定Loadできる形でセット
```
SHADER_TARGET < 45でも、以下のコマンドについては使用可能  
```
Texture2DMS<type,sampleCount> textureName  //MSAAテクスチャをsample番号指定Loadできる形でセット //MSAAサンプル数を明示的に指定
Texture2DArray<type> textureName  //普通のテクスチャアレイをセット
```
セットしたテクスチャは、LOAD_TEXTURE2D_MSAAなどのマクロでサンプリング可能。
[参考](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.core/ShaderLibrary/API/D3D11.hlsl#L146)  


---
# 型の変換
[参考](https://wlog.flatlib.jp/item/1072)  
値の変換をしたい場合は
```
uint theValue = (uint)floatValue;
```

そのままのbit値を 違う型として解釈したい場合は
```
uint theValue = asint(floatValue);

 // asint(), asfloat(), asuint()
```


---
# Consrevative Depth Test/Write関連
## 早期DepthTestの強制
[参考1](https://docs.microsoft.com/ja-jp/windows/win32/direct3dhlsl/sm5-attributes-earlydepthstencil), [参考2](https://www.gamedev.net/forums/topic/630218-conservative-depth-output-1-zw-depth-earlydepthstencil-and-early-z/), [参考3](https://forum.unity.com/threads/does-unity-2018-understand-the-sv_depthgreater-semantic.583135/)  
```
[earlydepthstencil]
float4 frag(){}
```
frag前のearly Z Test強制だけでなく、ZWriteも同時に行われる。  
後からfrag内でSV_DEPTHを出力したり、clipしても、早期に描き込んだZ値を改める事はできない。  

### 実例
自前のCommonOpaqueDepthOnly(不透明まとめ描き)シェーダに使ってみたら、逆に0.2ms程度遅くなったので やめといた。  
また、Depthをfragmentからいじらないシェーダーは、GPU判断でearlyDepthTest行われている事が多いと思われる。  


## Consrevative Depth
以下のようなSVをfragから出力することで、frag後には その値でDepthTest,Writeを行うが、early Z Testも行ってくれる。  
DxShaderModel5.0以降 = pragma4.5必要。  
VfxGraph - HDRPのパーティクルなどでも活用されている。あとUE4のWorldPossitionOffset（奥にしかシフトできない制限つきpixelオフセット）。  
```
float x :SV_DepthLessEqual
float x :SV_DepthGreaterEqual
```

### 制限
SV_POSITIONの入力に noperspective centroid （かつ MSAAsample系入力の不使用）を求められる。centroidだけでも通るが、はたして…。  
というか、基本的に vertexから出力したdepthと fragからposCS.zを出力した値が、厳密には一致しない。狂った値になる、という感じではないのだけど…。  
この為、Z Pre pass戦略（重いfragをZTest Equalで後書き）との合体がむずかしい。 noperspective centroidにするとワイヤー上だけdepth一致するのだけど。  
```
Interpolation mode for PS input position must be linear_noperspective_centroid or linear_noperspective_sample when outputting oDepthGE or oDepthLE 
and not running at sample frequency (which is forced by inputting SV_SampleIndex or declaring an input linear_sample or linear_noperspective_sample). 
```


---
# Add Custom Clipping Plane
参考1: https://msdn.microsoft.com/ja-jp/library/ee418355(v=vs.85).aspx ,  
[参考2](https://docs.microsoft.com/en-us/windows/desktop/direct3dhlsl/dx-graphics-hlsl-function-syntax)  
これで凹み投影デカール（でも手前の壁には隠れる）とかを表現しよう  

VertexOutputに
```
float  clip : SV_ClipDistance0; //[option]クリッピングプレーンの追加。
```
  
vertex shaderで
```
output.clip = dot(posWS, float4(0,1,0,1.5)); //[option]クリッピングプレーンの追加。
```
- プレーンの定義は： float4(x,y,z,l) //(x,y,z)ベクトルに鉛直な平面、原点からの距離l。
- 複数足せる。総数はUnityシェーダ記述より外で設定されてる。
- DirectX10以降