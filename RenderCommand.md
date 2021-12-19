//// RenderCommand関連 ////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# 描画時のソート設定
[参考](https://docs.unity3d.com/2020.3/Documentation/ScriptReference/Rendering.SortingCriteria.html)  
SortingCriteria、このbitmaskで rendererのソート設定を指定する。  
```
bit number
0 SortingLayer
1 RenderQueue
2 BackToFront
3 QuantizedFrontToBack
4 OptimizeStateChanges
5 CanvasOrder
6 RenderPriority
```

## SortingLayer
sorting layerに基づいてソート。  
レイヤー設定一覧からsorting layerリストを編集できる。Sprite,Canvas,Particleオブジェクトで、この所属を選べる。

## RenderQueue
マテリアルのRenderQueueに基づいてソート。  

## BackToFront
後ろから前の順でソート。半透明でよくあるやつ  

## QuantizedFrontToBack
雑に、前から後ろの順でソート。不透明でよくあるやつ  

## OptimizeStateChanges
マテリアル切り替えが減るように ほどほど調整。RenderQueueのように絶対的ではなく、Zソートされた上での子単位でしかやらないことに注意。  
combination of: static batching, lightmaps, material sort key, geometry ID

## CanvasOrder
キャンバス順でソート。  

## RenderPriority
SRPでrendererSupportsRendererPriority.trueにすると、meshRendererにTransparencyPriority設定欄がついて  
このフラグでソートできるぽいぞ。たぶん1と2の間。  
by renderer priority (if render queues are not equal)


## ex.一般的な不透明描画パス
```
CommonOpaque = SortingLayer | RenderQueue | QuantizedFrontToBack | OptimizeStateChanges | CanvasOrder
```

## ex.一般的な半透明描画パス
```
CommonTransparent = SortingLayer | RenderQueue | BackToFront | OptimizeStateChanges,
```

## ex.奥行きソートなしの不透明描画パス。
pre-depthバッファがある場などに有効。特に、ShaderのRenderQueueを細かく設定しておくことで、SRP batcher繋がり放題になる事が狙える
```
OpaqueWithoutFrontToBack = SortingCriteria.SortingLayer | SortingCriteria.RenderQueue | SortingCriteria.OptimizeStateChanges | SortingCriteria.CanvasOrder;
```


---
# ProfilingSampleルール
[参考](https://forum.unity.com/threads/profilingsample-usage-in-custom-srp.638941/)  
1. コマンドバッファ名と同名でBeginSample/EndSampleすること。
2. Sampleを入れ子にしたかったら、コマンドバッファも分けること。
3. BeginSample/EndSampleした直後に、コマンドバッファ実行をコンテクストに入力して、コマンドバッファをクリアすること。


---
# BeginRenderPass系
基本は[カスタムSRPサンプルのRenderPass使用例](https://github.com/cinight/CustomSRP/tree/master/Assets/SRP0802_RenderPass)、これだとシェーダ側のマクロがBRPのもののローカルコピーなので、  
Unity2021系でSRP側にBeginRenderPass系が入った実コードとして[URPのNativeRenderPass](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.universal/Runtime/NativeRenderPass.cs)と[SRP.core/Common.hlslのタイルメモリ取得系マクロ](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl#L205#L274)