//// RenderCommand関連 ////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# ProfilingSampleルール
[参考](https://forum.unity.com/threads/profilingsample-usage-in-custom-srp.638941/)  
1. コマンドバッファ名と同名でBeginSample/EndSampleすること。
2. Sampleを入れ子にしたかったら、コマンドバッファも分けること。
3. BeginSample/EndSampleした直後に、コマンドバッファ実行をコンテクストに入力して、コマンドバッファをクリアすること。

# BeginRenderPass系
基本は[カスタムSRPサンプルのRenderPass使用例](https://github.com/cinight/CustomSRP/tree/master/Assets/SRP0802_RenderPass)、これだとシェーダ側のマクロがBRPのもののローカルコピーなので、  
Unity2021系でSRP側にBeginRenderPass系が入った実コードとして[URPのNativeRenderPass](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.universal/Runtime/NativeRenderPass.cs)と[SRP.core/Common.hlslのタイルメモリ取得系マクロ](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl#L205#L274)