//// ShaderObject関連 /////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# RenderQueueの区分
```
----------------{0000}--------------------------------------------<
--BackGround----{1000}--------------------------------------------< 
--Geometry------{2000]--------------------------------------------< 一般的な 不透明オブジェクト
--AlphaTest-----{2450}--------------------------------------------< 一般的な パンチスルーオブジェクト
--Transparent---{3000}--------------------------------------------< 一般的な 半透明オブジェクト
--Overlay-------{4000}--------------------------------------------< Z無視のレンズフレアとか描く意図らしい。実質不使用
----------------{5000}--------------------------------------------<
```


---
# シェーダState系オプション
## 表裏カリング
```
Cull [_CullMode]

_CullMode =
	Off    //両面描画
	Front  //裏面のみ描画
	Back   //表面のみ描画
```

## デプス比較テスト
```
ZTest [_ZTest]

直接指定する際は以下のキーワード: Less | Greater | LEqual | GEqual | Equal | NotEqual | Always

_ZTest =
  |   |   |
0 [-------]	Disabled	 //常に描く
1          	Never
2 [-]		Less
3    [+]	Equal
4 [---+]	LesEqual	 //手前なら描く。デフォルト
5       [-]	Greater		 //奥なら描く
6 [-]   [-]	NotEqual
7    [+---]	GreaterEqual
8 [---+---]	Always
  |   |   |
```

## デプス描き込み
```
ZWrite [_ZWrite]

_ZWrite
	On    //Zバッファに描き込む
	Off   //Zバッファに描き込まない
```

## カラー合成
```
BlendOp [_BlendOp]
Blend   [_SrcBlend] [_DstBlend]

_BlendOp
	Add                        // SrcColor *_SrcBlend + DstColor *_DstBlend
	Sub (Subtract)             // DstColor *_DstBlend	- SrcColor *_SrcBlend
	RevSub (ReverseSubtract)   // SrcColor *_SrcBlend - DstColor *_DstBlend	
	Min                        // Min( SrcColor, DstColor)
	Max                        // Max( SrcColor, DstColor)

_BlendMode (= _SrcBlend, _DstBlend)
	Zero              // 0                  :  (0,0,0,0)
	One	              // 1                  :  (1,1,1,1)
	DstColor          // 画面の色	            : d(R,G,B,A)
	SrcColor          // シェーダ出力色       : s(R,G,B,A)
	OneMinusDstColor  // 反転 画面の色        :  (1,1,1,1) - d(R,G,B,A)
	SrcAlpha          // シェーダ出力アルファ  : s(A,A,A,A)
	OneMinusSrcColor  // 反転 シェーダ出力色	  :  (1,1,1,1) - s(R,G,B,A)
	DstAlpha          // 画面のアルファ       : d(A,A,A,A)
	OneMinusDstAlpha  // 反転 画面のアルファ   :  (1,1,1,1) - d(A,A,A,A)
	SrcAlphaSaturate  // 画面のアルファに空きがある範囲で加算ブレンドする時用  :  (f,f,f,1) ただしf=Min(As, 1-Ad)
	OneMinusSrcAlpha  // 反転 シェーダ出力アルファ  :	  (1,1,1,1) - s(A,A,A,A)
```

カラーとアルファそれぞれに 違うブレンド式を設定する
```
BlendOp [カラー_BlendOp] , [アルファ_BlendOp]
Blend   [カラー_SrcBlend] [カラー_DstBlend] , [アルファ_SrcBlend] [アルファ_DstBlend]
```

MRTごとにブレンド式を設定する
```
Blend 0 ～
Blend 1 ～
```


---
# Shader Lab
[参考](https://docs.unity3d.com/2020.3/Documentation/Manual/shader-writing.html)  
## シェーダーモデル設定
```
#pragma target 4.5
```
などと設定する。  

シェーダー内でターゲットに応じて分岐したいときは以下のように。
```
#if (SHADER_TARGET >= 45)
	return abs(ddx_fine(val));
#endif
```
  
4.5	= SM5.0	//PC新しめ  
3.5	=		//URPモバイル対応が　このへんターゲット  
[参考](https://docs.microsoft.com/en-us/windows/desktop/direct3dhlsl/d3d11-graphics-reference-sm5)  
  
target指定はSHADER_TARGETの足切りに使われるので、5.0を指定したからSHADER_TARGET=50になるわけではない。  
target 5.0指定　＞　PCでSM5.0でコンパイル　＞　SHADER_TARGET=45	など。  

### targetによって文法が変わる事もまれにある。
ex. target4.5以前だとMSAAテクスチャのバインド時にサンプル数の指定がいるが、4.5移行だといらない
```
Texture2DMS<float4, MSAA_SAMPLES> _CameraMsaaAttachment;

Texture2DMS<float4> _CameraMsaaAttachment;
```

## シェーダバリアント設定
### 基本の書き方
1．Properties {} 内で
```
[KeywordEnum(_ON,_SPECIAL)]
	_ALPHATEST("alpha option", Float) = 0
```
これを、  
Pass内で以下のように定義
```
#pragma multi_compile _ _ALPHATEST_ON _ALPHATEST_SPECIAL
 //または
#pragma shader_feature _ALPHATEST_ON
```
Shader内では以下のように使用
```
#ifdef _ALPHATEST_ON
#endif
 //または
#if defined(_ALPHATEST_ON)
#endif
```

２．Properties {} 内で
```
[Toggle(_ALPHATEST)] _EnableImpatient("Trace Impatiently", Float) = 1
```
これを同じく、Pass内で定義して、Shader内で使用
```
#ifdef _ALPHATEST
#endif
```

３．PassやShader側には静的分岐を用意しておき、Propertiesには書かず、直接Scriptから設定する  


## シェーダのState系オプションを、customEditorScriptを介さずに指定
[参考](https://github.com/microsoft/MixedRealityToolkit-Unity/blob/main/Assets/MRTK/StandardAssets/Shaders/MixedRealityStandard.shader)  
Properties {} 内で
```
[Enum(UnityEngine.Rendering.CullMode)] _CullMode("Cull Mode", Float) = 2                     // "Back"

[Enum(UnityEngine.Rendering.CompareFunction)] _ZTest("Depth Test", Float) = 4                // "LessEqual"
[Enum(DepthWrite)] _ZWrite("Depth Write", Float) = 1                                         // "On"
_ZOffsetFactor("Depth Offset Factor", Float) = 0                                             // "Zero"
_ZOffsetUnits("Depth Offset Units", Float) = 0                                               // "Zero"

[Enum(UnityEngine.Rendering.ColorWriteMask)] _ColorWriteMask("Color Write Mask", Float) = 15 // "All"
[Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend("Source Blend", Float) = 1                 // "One"
[Enum(UnityEngine.Rendering.BlendMode)] _DstBlend("Destination Blend", Float) = 0            // "Zero"
[Enum(UnityEngine.Rendering.BlendOp)] _BlendOp("Blend Operation", Float) = 0                 // "Add"

_RenderQueueOverride("Render Queue Override", Range(-1.0, 5000)) = -1

[Toggle(_STENCIL)] _Stencil("Enable Stencil Testing", Float) = 0.0
_StencilReference("Stencil Reference", Range(0, 255)) = 0
[Enum(UnityEngine.Rendering.CompareFunction)]_StencilComparison("Stencil Comparison", Int) = 0
[Enum(UnityEngine.Rendering.StencilOp)]_StencilOperation("Stencil Operation", Int) = 0
```
Pass内で
```
Cull[_CullMode]

ZTest[_ZTest]
ZWrite[_ZWrite]
Offset[_ZOffsetFactor],[_ZOffsetUnits]

ColorMask[_ColorWriteMask]
Blend[_SrcBlend][_DstBlend]
BlendOp[_BlendOp]

Stencil
{
	Ref[_StencilReference]
	Comp[_StencilComparison]
	Pass[_StencilOperation]
}
```


## HLSLコードがインクルードされる順序
Shader内にHLSLINCLUDEしたコードと、Shader/SubShader/Pass内のHLSLPROGRAMに書いたコードの順序関係について確認した。ついでに#pragma multi_compileとの順序関係も確認。

- そもそも同一HLSLスニペット内では、コード全部inline化した時の前後関係が そのまま順序となる。
.  
- HLSLINCLUDEとSubShaderの前後関係に関わらず、HLSLINCLUDEが先、HLSLPROGRAMが後、という順序で解釈される。
- 複数のHLSLINCLUDEがある場合、その前後関係に応じた順序で解釈される。
.  
- #pragma multi_compileで指定したフラグによるifdef分岐は、コードの前後関係に関わらず適用される。
　よって、SubShader内の#pragma multi_compileによる分岐は HLSLINCLUDE内のコードにも適用される。
.  
- #defineで指定したフラグによるifdef分岐は、コードの前後関係に応じて適用される。
　よって、SubShader内の#defineによる分岐は HLSLINCLUDE内のコードに適用されない。


---
# CBuffer
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
# vertexからfragmentに渡すプロパティ値の補完設定
[参考](https://docs.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-struct)  
ex.  
```
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
	+ ShaderModel4.1以降。MSAA必須。pixel中心ではなくsample点で値を取る。これ使った時点で、MSAAの全サンプルでpixelshaderが走る。(SV_SampleIndex読んだ時とかと同じように)。きれいー…なぜならuperSamplingだから。

## 複数の組み合わせが可能なもの
- centroid noperspective
	+ スクリーンスペース（centroid-adjusted affine）
- centroid linear
- sample noperspective
	+ MSAA強制
- sample linear


---
# SRP batcher対応ルール
- UnityPerDraw, UnityPerMaterialが仕様に沿って定義されていること。
- TextureやStructuredBufferは これに含まれない。ただしtextureSizeプロパティとかを使う場合 それらは含まれる。
- 以下の 1と2両方を満たしたときに、batcher対応からはずれる。
	+ 1. CBUFFER_START(UnityPerMaterial), CBUFFER_END　で挟んでいないところに定数が定義されていて
	+ 2. その定数がProperties内で同名定義されている
- materialPropertyBlock使った時、batcher対応からはずれる。
- 同一シェーダー内のPass違いでCBUFFERの内容変えたとき、batcher対応からはずれる。


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
# ちょっとマイナーなシェーダ命令
[この辺](https://docs.microsoft.com/en-us/windows/desktop/direct3dhlsl/dx-graphics-hlsl-semantics)も使える(DirectX固有のものを使った場合、Unityの機種互換サポートは死ぬ)。  
  
## VERTEXID_SEMANTIC //頂点番号
```
VertexInput  >> uint vertexID : VERTEXID_SEMANTIC
```
SRP系では、フルスクリーンポスプロを1triangleで描く時に、vertexIDを使ってGetFullScreenTriangleVertexPosition( vertexID, UNITY_RAW_FAR_CLIP_VALUE );などして、画面を覆うトライアングル座標とUVを提供している。

## FRONT_FACE_SEMANTIC //表裏判定
```
VertexOutput >> FRONT_FACE_TYPE cullFace : FRONT_FACE_SEMANTIC
```
シェーダ内で float isFrontVface = IS_FRONT_VFACE(cullFace, 1, 0); など分岐に使う

## SV_Coverage //Alpha to Coverge の出力先
Unityにおいては、shaderのPassにAlphaToMask Onをつけておくことで、AlphaToCoverge機能がオンになる。coverge値は出力アルファが使われる。  
さらにps_5_0だと、inoutにもできて、ピクセルシェーダ内で値を活用できるらしい  

## ddx_fine系
ddx_fine(), ddy_fine()、これらは	ddx(), ddy()の高精度版。  
SHADER_TARGET >= 45かつDirectX固有。Metalだと常にfineかも  
ddx_fineは横2ドットの偏微分、ddy_fineは縦2ドットの偏微分になる。精度2倍！（4倍ではない）  

## MSAAバッファのセット
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

