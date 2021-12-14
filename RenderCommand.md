//// RenderCommand関連 ////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# ProfilingSampleルール
[参考](https://forum.unity.com/threads/profilingsample-usage-in-custom-srp.638941/)
1. コマンドバッファ名と同名でBeginSample/EndSampleすること。
2. Sampleを入れ子にしたかったら、コマンドバッファも分けること。
3. BeginSample/EndSampleした直後に、コマンドバッファ実行をコンテクストに入力して、コマンドバッファをクリアすること。