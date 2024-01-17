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
# Unity DOTS Instancing
ECSのレンダリングパッケージEntity.Graphicsを使ってレンダリングする場合、またはBatchedRendererGroupを自分で使う場合、Unity2023以降で自動BRGをオンにした場合、
シェーダ側は この、DOTS Instancingパスを利用して描画する事になる。
このパスではMATRIX_Mも含めて個々のインスタンス描画情報がConstantBufferではなくGraphicsBufferに非同期アップロードされており、これをIDで拾いに行く必要がある。
といっても、基本的な対応はSRP core/Common.hlsl - Instancing.hlsl - DOTSInstancing.hlslあたりで定義されており、自作シェーダ側での対応はシンプルで、DOTS_INSTANCING_ON版のコンパイルを加えるだけ。VFX Graph利用を兼ねつつ書くとして以下のようになる。
```
#pragma multi_compile_instancing
#pragma instancing_options nolightprobe nolightmap nolodfade

#ifndef HAVE_VFX_MODIFICATION
 #pragma multi_compile _ DOTS_INSTANCING_ON
 #if UNITY_PLATFORM_ANDROID || UNITY_PLATFORM_WEBGL || UNITY_PLATFORM_UWP
  #pragma target 3.5 DOTS_INSTANCING_ON
 #else
  #pragma target 4.5 DOTS_INSTANCING_ON
 #endif
#endif
```

また、マテリアルの自前プロパティをDOTS Instancing時にインスタンスごと可変にするには、以下のようにプロパティ群を定義しておき、
```
#ifdef UNITY_DOTS_INSTANCING_ENABLED
 UNITY_DOTS_INSTANCING_START(MaterialPropertyMetadata)
   UNITY_DOTS_INSTANCED_PROP(type, name);
 UNITY_DOTS_INSTANCING_END(MaterialPropertyMetadata)
#endif
```
シェーダ内では以下のように値を読む必要がある。
```
hoge  = UNITY_ACCESS_DOTS_INSTANCED_PROP_WITH_DEFAULT(type, name);
```

## 独自RPでの使用
URP,HDRPでなく独自のRP内でDOTS INSTANCINGを利用する場合、以下のような定義も必要。
基本的には いつものPER_DRAWパラメータ参照マクロをDOTS_INSTANCINGで用意されるPROPに読み替える感じ。
あわせて、いつものGPU INSTANCINGにおけるINSTANCE_IDセットアップ用マクロにDOTS用の関数を追加している。
```
#ifdef UNITY_DOTS_INSTANCING_ENABLED
 UNITY_DOTS_INSTANCING_START(BuiltinPropertyMetadata)
    UNITY_DOTS_INSTANCED_PROP(float3x4, unity_ObjectToWorld)
    UNITY_DOTS_INSTANCED_PROP(float3x4, unity_WorldToObject)
    UNITY_DOTS_INSTANCED_PROP(float4,   unity_LODFade)
    UNITY_DOTS_INSTANCED_PROP(float4,   unity_RenderingLayer)
    UNITY_DOTS_INSTANCED_PROP(uint2,    unity_EntityId)
 UNITY_DOTS_INSTANCING_END(BuiltinPropertyMetadata)

 //.
 #define unity_LODFade               UNITY_ACCESS_DOTS_INSTANCED_PROP(float4,   unity_LODFade)
 #define unity_WorldTransformParams  LoadDOTSInstancedData_WorldTransformParams()
 #define unity_RenderingLayer        LoadDOTSInstancedData_RenderingLayer()

 //. Set up by BRG picking/selection code
 int unity_SubmeshIndex;
 #define unity_SelectionID UNITY_ACCESS_DOTS_INSTANCED_SELECTION_VALUE(unity_EntityId, unity_SubmeshIndex, _SelectionID)

 //.
 #if defined(MYRP_SETUP_INSTANCE_ID)
  #undef MYRP_SETUP_INSTANCE_ID
  #define MYRP_SETUP_INSTANCE_ID(input) {\
   UnitySetupInstanceID(UNITY_GET_INSTANCE_ID(input));\
   SetupDOTSVisibleInstancingData(); } //\
   //UNITY_SETUP_DOTS_SH_COEFFS; }
 #endif

#else
 #define unity_SelectionID _SelectionID

#endif
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
	float4 posOS	 : POSITION;	//頂点位置  //optional
	float3 normalOS  : NORMAL;		//法線ベクトル  //optional
	float4 tangentOS : TANGENT;		//タンジェントベクトル  //optional
	float4 uv0       : TEXCOORD0～;	//UV  //optional
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
	+ 無指定だとこれ。パースコレクト(posCS.w掛けておいてリニア補完後の値をposCS.wで割る感じ)ありのリニア補間
- nointerpolation
	+ 補間なし。たぶん、トライアングルを構成する最初の頂点?の値が そのまま来る。
- noperspective
	+ パース補正なし（つまりスクリーンスペース)のリニア補間
- centroid
	+ SV_POSITIONにサブピクセルのサンプリング点の値を出力する等。MSAA必須。トライアングルのエッジ（メッシュのエッジだけではない）でアンチかかる感じがあるが、精度あまいので、グラデがディザったりもするぞ。なので、その後ライン抽出とかやるとlinearより荒れる。
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
[参考](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.core/ShaderLibrary/API/D3D11.hlsl#L146)  


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


---
# ShaderからPrintf出力
シェーダからのデバッグ文字出力を、[自前実装する記事](https://therealmjp.github.io/posts/hlsl-printf/)が出たが、同じようにStructuredBufferを利用した手法がSRP.coreの[ShaderDebugPrint.hlsl](https://github.com/Unity-Technologies/Graphics/blob/2022.3/staging/Packages/com.unity.render-pipelines.core/ShaderLibrary/ShaderDebugPrint.hlsl)。  
ただしてテキストは4文字のタグに絞って提供されている。  
  
RenderPipeline側では、毎フレ[描画前](https://github.com/Unity-Technologies/Graphics/blob/2022.3/staging/Packages/com.unity.render-pipelines.universal/Runtime/ScriptableRenderer.cs#L1783#L1784)と[描画後](https://github.com/Unity-Technologies/Graphics/blob/2022.3/staging/Packages/com.unity.render-pipelines.universal/Runtime/UniversalRenderPipeline.cs#L403)にコードを挿入する。(URPならENABLE_SHADER_DEBUG_PRINTをdefineする)
```
//. Render()の、描画はじめる前に
ShaderDebugPrintManager.instance.SetShaderDebugPrintInputConstants(_Cmd, ShaderDebugPrintInputProducer.Get());
ShaderDebugPrintManager.instance.SetShaderDebugPrintBindings(_Cmd);

 〜

//. Render()の、最後のへんで
ShaderDebugPrintManager.instance.EndFrame();
```
  
シェーダ側では、ShaderDebugPrint.hlslをincludeして、ShaderDebugPrint()関数を利用する。文字は4文字まで、ShaderDebugTag()でintにパック。  
PosCS等で絞り込んで 特定ピクセルのみデバッグ出力する形にするよう注意。ShaderDebugPrintMouseOver()関数でマウス直下のピクセルのみデバッグ出力も可能。
```
if(all(int2(input.posCS.xy) == int2(100, 100)))
	ShaderDebugPrint(ShaderDebugTag('C','o','l'), output.color);

ShaderDebugPrintMouseOver(int2(input.posCS.xy), ShaderDebugTag('E','m','i', 't'), output.emit);

```

