///[ SRPの(Native)RenderPassについて ]  
/////////////////////////////////////////////////////////////  
  
# (Native)RenderPassとは
モバイルや AppleシリコンPCのGPUは、TBDR描画(画面を小さいタイルに分けて描いていく)という特徴を持つ。これらプラットフォームにおいては、Metal,Vulkanといった描画APIで、オンタイルメモリを活用する事で、高度な描画の低負荷,低電力消費な実現が期待できる。  
Unityの[RenderPass](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Rendering.ScriptableRenderContext.BeginScopedRenderPass.html)系APIは、 この目的で用意されている。  


---
# 実用性について
## 非対応プラットフォーム
オンタイルメモリ運用に対応していないOpenGL,DirectX およびTBDRでないPC上のVulkanでは、通常のMultiRenderTarget描画にフォールバックした上で、基本的には同等の描画結果が得られる。  
  
## 実装コスト
- 描画コマンドのうち、SetRenderTarget周りが大きく異なる
- シェーダコードのうち、RenderTextureのロード周りが異なる。アタッチされたMRTのスロットではなくPassで定義した番号でロードするなどクセ強めなので、マクロでカバーしたい。なお、シェーダ出力は普通のMRTと一緒
- MetalとVulkanでMSAAのサブピクセルサンプリング仕様が異なるなど、プラットフォーム差は びみょうに残る
  
## URP-RenderGraph上での実用状況
- RenderGraphのリソース管理部分で (かなり保守的にだが)NativeRenderPass対応が行われる。シェーダコードは自前での用意が必要
- URPの定義した用語 ScriptableRenderPassと名前が被っていて、ややこしい


---
# 実装イメージ
  
C#側
```
//. パス内で使用するRenderTexture群を定義しておく
///emitカラーとマスク,デプスステンシルのRTを使用するが オンタイルメモリ上で運用し、emitをMSAAリゾルブした結果だけがRenderTextureとして保存される
_Rad.emit = new AttachmentDescriptor(gf_HDR_Linear);
_Rad.mask = new AttachmentDescriptor(gf_LDR_Linear);
_Rad.depstcl = new AttachmentDescriptor(gf_Depth);

_Rad.emit.ConfigureClear(k_ClearColor, 1.0f, 0x00);
_Rad.emit.ConfigureResolveTarget(_RtId.emit);
_Rad.depcopy.ConfigureClear(k_ZeroColor, 1.0f, 0x00);
_Rad.mask.ConfigureClear(k_ClearColor, 1.0f, 0x00);

_RpRads = new NativeArray<AttachmentDescriptor>(3, Allocator.Persistent, NativeArrayOptions.UninitializedMemory);
_RpRads[0] = _Rad.emit;
_RpRads[1] = _Rad.mask;
_RpRads[2] = _Rad.depstcl;

//. SubPassごとのMRT使用予定を定義しておく
/// サブパス１でemitとmaskに描き、サブパス２でmaskを読みつつemitに描く。
_RpOneOut = new NativeArray<int>(2, Allocator.Persistent, NativeArrayOptions.UninitializedMemory);
_RpPreOut[0] = 0;
_RpPreOut[1] = 1;

_RpTwoIn = new NativeArray<int>(3, Allocator.Persistent, NativeArrayOptions.UninitializedMemory);
_RpTwoIn[0] = 1;

_RpTwoOut = new NativeArray<int>(1, Allocator.Persistent, NativeArrayOptions.UninitializedMemory);
_RpTwoOut[0] = 0;

//. 描画の実行
using (context.BeginScopedRenderPass(k_Rt_Width, k_Rt_Height, k_Rt_Msaa, _RpRads, 2))
{
	//. サブパス１
	using (context.BeginScopedSubPass(_RpOneOut, false))
	{
		_Cmd.DrawRendererList(rl_One);
		context.ExecuteCommandBuffer(_Cmd);
		_Cmd.Clear();
	}

	//. サブパス２
	using (context.BeginScopedSubPass(_RpTwoOut, _RpTwoIn))
	{
		_Cmd.DrawRendererList(rl_Two);
		context.ExecuteCommandBuffer(_Cmd);
		_Cmd.Clear();
	}
}
```
Shader側
```
//. オンタイルメモリ ロード用マクロの定義
#if defined(PLATFORM_SUPPORTS_NATIVE_RENDERPASS)
 #define FB_INPUT_HALF_MS(idx) cbuffer hlslcc_SubpassInput_H_##idx { half4 hlslcc_fbinput_##idx[8]; }
 #define LOAD_FB_INPUT_MS(idx, sampleIdx, v2fname) hlslcc_fbinput_##idx[sampleIdx]
#else
 #define FB_INPUT_HALF_MS(idx) Texture2DMS<float4> _UnityFBInput##idx; float4 _UnityFBInput##idx##_TexelSize
 #define LOAD_FB_INPUT_MS(idx, sampleIdx, v2fname) _UnityFBInput##idx.Load(uint2(v2fvertexname.xy), sampleIdx)
#endif

//. サブパス２のシェーダ内で、オンタイルメモリのロード
FB_INPUT_HALF_MS(0);
FB_INPUT_HALF_MS(1);

half4 emit = LOAD_FB_INPUT_MS(0, 0, posCS_XY);
half4 mask = LOAD_FB_INPUT_MS(1, 0, posCS_XY);

```


---
# 関連リンク(OLD)
・[カスタムSRPサンプルのRenderPass使用例](https://github.com/cinight/CustomSRP/tree/master/Assets/SRP0802_RenderPass)、ただ、これはシェーダ側のマクロがBRPのもののローカルコピーなので、  
Unity2021系でSRP側にBeginRenderPass系が入った実コードとして[URPのNativeRenderPass](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.universal/Runtime/NativeRenderPass.cs)と[SRP.core/Common.hlslのタイルメモリ取得系マクロ](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl#L252#L390)