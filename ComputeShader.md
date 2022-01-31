//// ComputeShader ///////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# スレッド数
テクスチャ一枚(width,height)描くのを手分けするとして、  
.  
  ディスパッチ側　　　　　         シェーダー側  
 [width/8 x height/8 x 1]    x    [numthreads(8 x 8 x 1)]  
.  
 …みたいな、6次元のスレッド数が起動する。  
 シェーダー上で、メモリ共有したり同期とったりできるのは、シェーダー側で指定したスレッド数の範囲まで。  
.  
シェーダ側では、現在のスレッドのIDを各種	semanticsで取得できる  
```
uint3 DTid : SV_DispatchThreadID	//全体通しの スレッド番号		(x,y,z) = (0～[width-1],		0～[height-1],		0)
uint3 Gid  : SV_GroupID				//ディスパッチ側の スレッド番号	(x,y,z) = (0～[dispatchX -1],	0～[dispatchY -1],	0)
uint3 GTid : SV_GroupThreadID		//シェーダー側の スレッド番号	(x,y,z)	= (0～[blocksizeX-1],	0～[blocksizeY-1],	0)
uint  GI   : SV_GroupIndex			//シェーダー側の スレッド番号	(n)		= (0～[blocksizeX*blocksizeY -1])
```


---
# ex.テクスチャ一通り なにか加工する
- ターゲットデータの大きさは(width, height)
- CPUから、csを(dispatchX, dispatchY, 1)でdispatchする。
- csは[numthreads(blocksizeX, blocksizeY, 1)]のシェーダー。

・基本は、  
　　width  = dispatchX * blocksizeX  
　　height = dispatchY * blocksizeY  
　になるよう dispatchX,dispatchYを調整する。  
.  
## C#側
```
ComputeShader cs = (ComputeShader)Resources.Load("CsExtendDistanceField");
	//CSファイルの参照にシェーダーパスは使えない。SerializeFieldで持っておくか、Resourcesフォルダに入れておくとか。

cs.SetFloat("width", texRt.width);
cs.SetTexture(0, "cSdfInTex",  cSdfInTex);
	//テクスチャセット時にkernelIndexの指定がある。とりあえずCSMain使ってる分には0番でいいような、ちゃんとカーネル番号取得しておいても良いような。

cs.Dispatch(0, texRt.width / 8, texRt.height / 8, 1);
	//そして実行。kernelIndexの指定と、3次元のスレッド数(ディスパッチ側)の指定。
```

 ## Shader側
 ```
Texture2D cSdfOutTex;
float     width, height;

RWTexture2D<float4> Result; //出力先

#define blocksize 8
#define groupthreads (blocksize*blocksize)
groupshared float4 accum[groupthreads];		//スレッド間共有メモリ >> accum[GI] >>float5にしてバンク衝突を避けた方が2倍速いぞ

[numthreads(blocksize, blocksize, 1)]
void CSMain(
uint3 DTid : SV_DispatchThreadID	//全体通しの	    スレッド番号(x,y,z) = (0～[width-1],	0～[height-1],   0)
uint3 Gid  : SV_GroupID				//ディスパッチ側の	スレッド番号(x,y,z) = (0～[width/8-1],	0～[height/8-1], 0)
uint3 GTid : SV_GroupThreadID		//シェーダー側の	スレッド番号(x,y,z)	= (0～7,			0～7,			 0)
uint  GI   : SV_GroupIndex			//シェーダー側の	スレッド番号(n)		= (0～63)
){

	GroupMemoryBarrierWithGroupSync();	//グループ内の全スレッドを ここで同期待ち
}
```

---
# 実際には、CSで画像処理するなら、mortonレイアウトを使うのが良い
SRP.coreの[DecodeMorton2D](https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.core/ShaderLibrary/SpaceFillingCurves.hlsl#L62#L65)とか利用しよう。  

## Shader側
```
#define blocksize 16

[numthreads((blocksize*blocksize)), 1, 1)]
void CSMain(uint GTid : SV_GroupThreadID, uint2 Gid : SV_GroupID)
{ 
  //.
  uint2 pixelPosSCS = Gid * blocksize + DecodeMorton2D(GTid);
  ...

```