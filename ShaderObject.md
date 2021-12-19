//// ShaderObject関連 /////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# Shader Lab
ShaderObjectは\*.shaderファイルとして用意し、そのコードはShader Labを介して各種描画API用シェーダコードに変換される
[参考](https://docs.unity3d.com/2020.3/Documentation/Manual/shader-writing.html)  


---
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

## デプステスト
```
ZTest [_ZTest]

_ZTest =
直接指定する際は以下のキーワード: Less | Greater | LEqual | GEqual | Equal | NotEqual | Always
  |   |   |
0 [-------]	Disabled	 //常に描く
1          	Never
2 [-]		Less
3    [+]	Equal
4 [---+]	LEqual	 //手前なら描く。デフォルト //LesEqual
5       [-]	Greater		 //奥なら描く
6 [-]   [-]	NotEqual
7    [+---]	GEqual //GreaterEqual
8 [---+---]	Always
  |   |   |
```

## デプス描き込み
```
ZWrite [_ZWrite]

_ZWrite = 
	On    //Zバッファに描き込む
	Off   //Zバッファに描き込まない
```

## カラー合成
```
BlendOp [_BlendOp]
Blend   [_SrcBlend] [_DstBlend]

_BlendOp = 
	Add                        // SrcColor *_SrcBlend + DstColor *_DstBlend
	Sub (Subtract)             // DstColor *_DstBlend	- SrcColor *_SrcBlend
	RevSub (ReverseSubtract)   // SrcColor *_SrcBlend - DstColor *_DstBlend	
	Min                        // Min( SrcColor, DstColor)
	Max                        // Max( SrcColor, DstColor)

_BlendMode (= _SrcBlend, _DstBlend) =
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

カラーとアルファそれぞれに 違うブレンド式を指定する
```
BlendOp [カラー_BlendOp] , [アルファ_BlendOp]
Blend   [カラー_SrcBlend] [カラー_DstBlend] , [アルファ_SrcBlend] [アルファ_DstBlend]
```

MRTごとにブレンド式を指定する
```
Blend 0 ～
Blend 1 ～
```

ex.不透明
ex.半透明(ストレート)
ex.半透明(PreMultiply利用により半透明と加算を両立)
ex.加算
ex.乗算
ex.乗算2倍


## ステンシルテスト
```
Stencil
{
	Ref [_StencilReference]
	Readmask [_StecilBitmask_Read]	//optional 
	Writemask [_StencilBitmask_Write]	//optional 
	Comp [_StencilComparison]
	Pass [_StencilOperation_Pass]
	Fail  [_StencilOperation_Fail]	//optional
	ZFail [_StencilOperation_ZFail]	//optional
}

_StencilReference = 
適用する数値。リファレンス値 (8bit bitmask, 0 - 255)

_StencilBitmask_Read = 
リファレンス値を描き込み先のステンシル値と比較する際に適用するビットマスク (8bit bitmask, 0 - 255)

_StencilBitmask_Write = 
リファレンス値を描き込む際に適用するビットマスク (8bit bitmask, 0 - 255)

_StencilComparison = 
リファレンス値と描き込み先のステンシル値を比較する為の演算子。
比較(ステンシルテスト)をパスした場合のみ、fragmentシェーダを実行し、カラー,depthバッファへの描き込みを行う。
また、ステンシルバッファについては、同テストの結果に応じてPass,Fail,ZFailで指定した描き込み操作を行う。
  |   |   |
0 [-------]	Disabled
1          	Never
2 [-]		Less
3    [+]	Equal
4 [---+]	LEqual	
5       [-]	Greater
6 [-]   [-]	NotEqual
7    [+---]	GEqual
8 [---+---]	Always //常に描く。デフォルト
  |   |   |

 _StencilOperation = 
ステンシルテストの結果に応じて、描き込み先のステンシル値に、ここで指定した描き込み操作を行う。
比較(ステンシルテスト)をパスした場合はPass、しなかった場合はFail、ステンシルテストはパスしたがデプステストにはパスしなかった場合はZFailの操作を行う。
0 Keep //描き込まない。デフォルト
1 Zero //0でクリアする
2 Replace //リファレンス値を描き込む
3 IncrSat //描き込み先の値を1大きくする。255なら255のまま //IncrementSaturate
4 DecSat //描き込み先の値を1小さくする。0なら0のまま //DecremenetSaturate
5 Invert //描き込み先の数値をbitmaskとして全bit反転する
6 IncWrap //描き込み先の値を1大きくする。255なら0にする //IncrementWrap
7 DecWrap //描き込み先の値を1小さくする。0なら255にする //DecrementWrap
```

ポリゴン面が表か裏かに応じて、違う内容のステンシルテストを行う
```
表面  CompFront, PassFront, FailFront, ZFailFront
裏面  CompBack, PassBack, FailBack, ZFailBack 
```

ex.ステンシルテストは行わず、ステンシルバッファには4を描き込む。
```
Stencil
{
	Ref 4
	Pass Replace
}
```

ex.描き込み先のステンシル値が4だった場合のみ、当該ピクセルに描き込む。
```
Stencil
{
	Ref 4
	Comp Equal
}
```

ex.描き込み先の第3bitフラグが立っていた場合のみ、当該ピクセルに描き込む。他のビットの状態は問わない
```
// value   4 = 0000 0100
// bitmask 4 = xxxx x1xx

Stencil
{
	Ref 4
	Readmask 4
	Comp Equal
}
```


---
# シェーダーモデル設定
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


---
# シェーダバリアント設定
## 基本的な書き方
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


---
# HLSLコードがインクルードされる順序
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
