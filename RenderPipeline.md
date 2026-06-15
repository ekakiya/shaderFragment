///[ SRPのRenderPipelineについて ]  
/////////////////////////////////////////////////////////////  
  
# ScriptableRenderPipelineの動作
RenderPipeline継承クラスを作り、描画パイプラインとしてセットすると、毎フレーム LateUpdate後RenderThread同期前のタイミングで、Render()コマンドが呼ばれる。  
コールでは 抽象的な描画処理コマンド群を積めるScriptableRenderContextと シーン内cameraのリストが渡されるので、contextへ   好きにコマンドを積んでsubmitする。このコマンドを積む処理は基本MainThreadで実行され、その後、積んだコマンド群が RenderThreadで逐次実行される。  
  
なお、自作したRP継承クラスを 描画パイプラインとしてセットするには、RenderPipelineAsset継承クラス(これはScriptableObjectなのでAssetを作成できる)  のCreate()コマンドでRP継承クラスのインスタンスを返すよう実装し、このRPAssetをProjectSettingsにセットすればよい。  

常用テクスチャなどリソースとして使いたいアセットは RenderPipelineAssetに直接持たせず、RenderPipelineResourcesみたいな名前のScriptableObjectを作って そちらを参照する形を取る。  


# Render()の注意点
- editor上では、綺麗に毎フレーム1回 Render()が呼ばれる訳ではない。Play中のGameビューからは (ランタイム上と同様に)綺麗なRenderコールが来るが、それ以外にSceneビュー, 選択中Cameraのプレビュー, Inspecter上のマテリアルやモデルプレビュー,マテリアルのアイコン 等から、不定期に、各々のRenderコールが来る。
	+ もし、毎フレーム実行のつもりで Renderコール每にComputeShaderを1回実行する実装をした場合、上記Renderコールの影響で 謎の実行結果になり、悩むことに・・
	+ なので、自作RPの実装では、いったんGameビューカメラ(camera.cameraType == CameraType.Game)以外のコールを 切り分けた形で始めると、混乱が少ない
- 毎フレーム実行されるので、丁寧にAllocを避ける。頻用するStringsもShader.PropertyToIDでIDの形で持っておく等


# ScriptableRenderContextについて
細かくMeshRenderer単位のコントロールは行わない。レンダーターゲットのセット／クリア、Renderer群を指定カメラでカリング、Renderer群を指定フィルタにかけて描画、など  
大枠の処理手順をScriptableRenderContextに記述し Submitする。  
記述に際しては、BiRP旧来からある CommandBufferによる描画コマンドリストも利用できる（context.ExecuteCommandBuffer(cmd)で差し込む形）  

## 一般的な描画の流れ 概要
```
//. カリング
camera.TryGetCullingParameters(out cullParams) // カメラからカリング設定を取得
cullResult = context.Cull(ref cullParams) // カリング実行

//. 描画対象のリスト作成
rList = context.CreateRendererList(rListParams(cullResult)) // カメラ視野内にあるMeshRendererのリストを作成

//. カメラ座標系などセット
context.SetupCameraProperties(camera) // BiRP同様のカメラMATRIXやClipPlane設定をGPUにアップロード。他の こまごま処理も実行

//. 描画ターゲットにRenderTextureをセット
cmd.SetRenderTarget(BuiltinRenderTextureType.CameraTarget) // 描画ターゲットに、camera設定に基づきスクリーンバックバッファ等を指定

//. 描画実行
cmd.DrawRendererList(rList) // 描画対象のリストについて、描画実行
context.ExecuteCommandBuffer(cmd)
cmd.Clear()
```
## 一般的な描画の流れ　カスタム設定箇所
```
//. カリング
camera.TryGetCullingParameters(out cullParams) 
cullParams.cullingOption |= CullingOptions.DisablePerObjectCulling
	// カリング設定をカスタム。ex.ライトや環境マップと各Objectの接触判定を削除
cullResult = context.Cull(ref cullParams)

/. 描画対象のリスト作成
sortingSettings = new SortingSettings(camera) { criteria = SortingCriteria.RenderQueue } // ソート順の設定をカスタム
drawingSettings = new DrawingSettings(shaderTag, sortingSettings)
	// バッチ,GPU Instancing, MeshRenderer単位の取得データなどの設定をカスタム
filteringSettngs = new FilteringSettings(rendeerQueueRange) // 描画するレンダーキューの範囲をカスタム
rListParams = new RendererListParams(cullResult, drawingSettings, filteringSettings)
rList = context.CreateRendererList(rListParams)
rLists.Clear()
rLists.Add(rList)
context.PrepareRendererListsAsync(rLists)
	// 非同期でRendererList作成を即時開始。この結果を元に 不要な描画パスを切るなど出来るが、実測値的に お得だった事はあまりない

//. カメラ座標系などセット
context.SetupCameraProperties(camera) // Matrix系を自前でやっていれば このコマンドを省略しても動作するが、不安は残る
dataPerCameraに自前カメラマトリックスなどを詰める
cmd.SetBufferData(graphicsBuffer, dataPerCamera); // 詰めた値をgraphicsBufferにアップロード
cmd.SetGlobalConstantBuffer(graphicsBuffer, ShaderId_BufferPerCamera, 0, sizeOfGraphicsBuffer)
	// そのgraphicsBufferをconstantBufferとしてセット。DrawCallごとのセットアップ負荷を削減

//. 描画ターゲットにRenderTextureをセット
cmd.SetRenderTarget(BuiltinRenderTextureType.CameraTarget) 

//. 描画実行
cmd.DrawRendererList(rList) 
context.ExecuteCommandBuffer(cmd)
cmd.Clear()
```
## NativeRenderPassを使った、描画の流れ 概要
```
//.事前に、各描画パス,描画サブパスの描画ターゲットをセットアップしておく
radColor = new AttachmentDescriptor(gf_HDR_Linear) // HDRバッファを カラー描画に使う
radMask = new AttachmentDescriptor(gf_LDR_Linear) // LDRバッファを マスク用途に使う
radDepStcl = new AttachmentDescriptor(gf_Depth) // デプスステンシルバッファも用意
radColor.ConfigureResolveTarget(renderTexture)
	// カラー描画バッファをMSAAリゾルブしてから、実際のRenderTextureに出力する。他はオンタイルメモリ完結でGPUメモリ書き込みを省く
rads = new NativeArray<AttachmentDescriptor>(3, Allocator.Persistent, NativeArrayOptions.UninitializedMemory)
rads[0] = radColor; rads[1] = radMask; rads[2] = radDepStcl
rPassOneOut = new NativeArray<int> 〜 [0]=0 [1]=1 // サブパス１では、カラーとマスク両方に描く
rPassTwoIn = new NativeArray<int> 〜 [0]=1 // サブパス２では、マスクを読んで利用する
rPassTwoOut = new NativeArray<int> 〜 [0]=0 // サブパス２では、カラーに描く

//. カリング
camera.TryGetCullingParameters(out cullParams)
cullResult = context.Cull(ref cullParams)

//. 描画対象のリスト作成
rList = context.CreateRendererList(rListParams(cullResult))

//. カメラ座標系などセット
context.SetupCameraProperties(camera)

//. 事前にセットアップしてある 描画パスを開始
using (context.BeginScopedRenderPass(rt_width, rt_Height, rt_Msaa, rads, 2)){

//. サブパス1 描画実行
	using (context.BeginScopedSubPass(rPassOneOut, false)){
		cmd.DrawRendererList(rList)
		context.ExecuteCommandBuffer(cmd)
		cmd.Clear()
	}

//. サブパス2 描画実行
	using (context.BeginScopedSubPass(rPassTwoOut, rPassTwoIn)){
		cmd.DrawRendererList(rList)
		context.ExecuteCommandBuffer(cmd)
		cmd.Clear()
	}
}
```