//// Mesh関連 ////////////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# VertexBufferの設定
Mesh生成時、VertexBufferの設定方法が何種類かある。

## Mesh.uv系
実は[これ](https://docs.unity3d.com/ScriptReference/Mesh-uv.html)が一番レガシーで、制限が多い。  
uvがfloat2しかセットできない等。

## Mesh.SetUvs系
今、順当に使うならこれだと思う。[SetUvs](https://docs.unity3d.com/ScriptReference/Mesh.SetUVs.html)はuvにfloat2〜float4を明示的に指定できる。  
[SetColors](https://docs.unity3d.com/ScriptReference/Mesh.SetColors.html)はColor32をセットすれば旧来の4byteカラー精度で、Colorsをセットすればfloat4精度の値を指定できる。  
ランタイム上で効率的に更新する用途であれば、NativeArrayもセットできる。

## Mesh.SetVertexBufferData系
[SetVertexBufferParams](https://docs.unity3d.com/ScriptReference/Mesh.SetVertexBufferParams.html)でバッファのレイアウトを指定、[SetVertexBufferData](https://docs.unity3d.com/ScriptReference/Mesh.SetVertexBufferData.html)で中身をセット。  
いちばんストレートな感じがするが、プラットフォーム差の吸収とか手動でやらないといけない気がして、未検証の為、実験的にしか利用していない。


# Meshのファイナライズ
雰囲気で生成したメッシュを、triangle stripとかいい感じにして欲しい時は[Mesh.Optimize](https://docs.unity3d.com/ScriptReference/Mesh.Optimize.html)。  
メッシュが完成したら[Mesh.UploadMeshData(true)](https://docs.unity3d.com/ScriptReference/Mesh.UploadMeshData.html)でGPUデータを更新&CPUデータを削減する。カスタムインポータでMewshアセット作っている時も、これによってisReadableフラグを切ること。