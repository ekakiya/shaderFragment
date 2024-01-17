//// Texture関連 //////////////////////////////////////////////////////////////
//				
//////////////////////////////////////////////////////////////////////////////

# RenderTextureFormat
## GraphicsFormat指定でRenderTexture作成
SRP.coreに用意された RTHandleクラスを使う事で、間接的に この手法を取れるが、他にも色んな機能が付いてきてしまう。  
シンプルにGraphicsFormat指定で生成する場合は以下。  
```diff
! ex.
RenderTexture _RtColor = new RenderTexture(k_RtColor_width, k_RtColor_height, (int)DepthBits.None, GraphicsFormat.R16G16B16A16_SFloat)
{
	hideFlags = HideFlags.HideAndDontSave,
	volumeDepth = 1,
	filterMode = FilterMode.Bilinear,
	wrapMode = TextureWrapMode.Clamp,
	dimension = TextureDimension.Tex2D,
	enableRandomWrite = false,
	useMipMap = false,
	autoGenerateMips = false,
	anisoLevel = 1,
	mipMapBias = 0f,
	antiAliasing = (int)MSAASamples.MSAA4x,
	bindTextureMS = true,
	useDynamicScale = false,
	//vrUsage = VRTextureUsage.None,
	memorylessMode = RenderTextureMemoryless.None,
	name = CoreUtils.GetRenderTargetAutoName(k_RtColor_width, k_RtColor_height, 1, GraphicsFormatUtility.GetRenderTextureFormat(GraphicsFormat.R16G16B16A16_SFloat), 
	"ColorBuffer", mips: false, enableMSAA: true, msaaSamples: MSAASamples.MSAA4x)
};
_RtColor.Create();
```

## PC Standaloneにおける フォーマット読み替え
基本的に、RenderTexture作成時はGraphicsFormat使用で統一したいが、アセットとしてRenderTextureを用意する時は、どうしても こちらを使う事になってしまうので…。
```
ARGB32 = 0,			// R8G8B8A8_SRGB
Depth = 1,			// DEPTH_AUTO
ARGBHalf = 2,		// R16G16B16A16_SFLOAT
Shadowmap = 3,		// SHADOW_AUTO
RGB565 = 4,			// B5G6R5_UNORM_PACK16
ARGB4444 = 5,		// B4G4R4A4_UNormPack16 is not supported. Fallbacks to R8G8B8A8_UNorm format on this platform.
ARGB1555 = 6,		// B5G5R5A1_UNIFORM_PACK16
Default = 7,		// R8G8B8A8_SRGB

ARGB2101010 = 8,	// A2B10G10R10_UNORM_PACK32
DefaultHDR = 9,		// R16G16B16A16_SFLOAT ★
ARGB64 = 10,		// R16G16B16A16_UNORM
ARGBFloat = 11,		// R32G32B32A32_SFLOAT
RGFloat = 12,		// R32G32_SFLOAT
RGHalf = 13,		// R16G16_SFLOAT
RFloat = 14,		// R32_SFLOAT
RHalf = 15,			// R16_SFLOAT

R8 = 16,				// R8_UNORM ★
ARGBInt = 17,			// R32G32B32A32_SINT
RGInt = 18,				// R32G32_SINT
RInt = 19,				// R32_SINT
BGRA32 = 20,			// ? エラーなし、空欄になる
// kRTFormatVideo = 21,
RGB111110Float = 22,	// B10G11R11_UFLOAT_PACK32
RG32 = 23,				// R16G16_UNORM

RGBAUShort = 24,		// R16G16B16A16_UINT
RG16 = 25,				// R8G8_UNORM
BGRA10101010_XR = 26,	// A10R10G10B10_XRSRGBPack32 is not supported. Fallbacks to R16G16B16A16_UNorm format on this platform. 
BGR101010_XR = 27,		// R10G10B10_XRSRGBPack32 is not supported. Fallbacks to R16G16B16A16_UNorm format on this platform.
R16 = 28,				// R16_UNORM

★・・・R8G8B8A8_UNORMが無い・・・。
```


---
# Textureインポート時に何かやる
[参考](https://github.com/keijiro/unity-dither4444/blob/master/Assets/Editor/TextureModifier.cs)
## 基本
```diff
! ex.
public sealed partial class TextureImportProcessor : AssetPostprocessor
{
	void OnPreprocessTexture()
	{
		TextureImporter ti = assetImporter as TextureImporter;

		if (ti.assetPath.Contains("KeyName"))
		{
			ti.textureFormat = TextureImporterFormat.RGBA32;
			PreProcess_KeyName(ref ti);
		}
	}

	void OnPostprocessTexture(Texture2D texture)
	{
		TextureImporter ti = assetImporter as TextureImporter;

		if (ti)
		{
			if (ti.assetPath.Contains("KeyName"))
			{
				PostProcess_KeyName(ref ti, ref texture);
				EditorUtility.CompressTexture(texture, TextureFormat.BC7, TextureCompressionQuality.Best);
			}
		}
	}
}
```

## ComputeShader利用
Resourcesにcomputeファイル置いて使っています…。  
```diff
! ex.
ComputeShader cs = (ComputeShader)Resources.Load("CsExtendDistanceField");

RenderTexture _RtCompute = new RenderTexture(texWidth, texHeight, 0, RenderTextureFormat.ARGBFloat);
_RtCompute.enableRandomWrite = true;
_RtCompute.Create();

cs.SetTexture(0, "InputTex", texture);
cs.SetTexture(0, "ResultTex", _RtCompute);
cs.Dispatch(0, texWidth / 16, texHeight / 16, 1);

RenderTexture nowRt = RenderTexture.active;
RenderTexture.active = _RtCompute;
texture.ReadPixels(new Rect(0, 0, texWidth, texHeight), 0, 0);
texture.Apply();

RenderTexture.active = nowRt;
_RtCompute?.Release();
```


---
# Mipmap限界レベル設定
[参考1](https://github.com/Unity-Technologies/UnityCsReference/blob/master/Runtime/Export/Graphics/Texture.cs#L829), [参考2](https://forum.unity.com/threads/limiting-the-amount-of-mipmap-levels.650011/#post-5089640)  
TextureのMipmap限界レベルを、生成時にスクリプトから設定できる。
```
new Texture2D(width:256, height:256, textureFormat:TextureFormat.ARGB32, mipCount:3, linear:true)
```

```diff
! ex.
public static Texture2D CreateLimitedMipmapsTexture(Texture2D sourceTexture, TextureFormat format, int mipmapCount, bool isLinear, int anisoLevel, FilterMode filterMode)
{
	var texturePath = AssetDatabase.GetAssetPath(sourceTexture);
         
	var limitedMips = new Texture2D(sourceTexture.width, sourceTexture.width, format, mipmapCount, isLinear);
	limitedMips.anisoLevel = anisoLevel;
	limitedMips.filterMode = filterMode;
         
	for (var i = 0; i < mipmapCount; i++)
	{
		Graphics.CopyTexture(sourceTexture, 0, i, limitedMips, 0, i);
	}
         
	var outputPath = $"{texturePath.Replace($"{sourceTexture.name}.png", $"{sourceTexture.name}__LimitedMipCount-{mipmapCount}.asset")}";
         
	AssetDatabase.CreateAsset(limitedMips, outputPath);
	AssetDatabase.SaveAssets();
         
	return AssetDatabase.LoadAssetAtPath<Texture2D>(outputPath);
 }
```

# Gather
きっちり特定pixelを示すUVでGatherした場合、集められる4サンプルは、当該ピクセルを左下にした2x2 pixel。
下の図でいうaチャンネルがUV直下の値となる。

r | g
--+--
a | b
