# StudioTextAtlasFix

Patched **KK_StudioTextPlugin** (Koikatsu studio text plugin, [jim60105](https://github.com/jim60105/KK)'s version) that fixes the long-standing bug where **CJK text objects randomly disappear** from studio scenes.

## The bug

The plugin binds the font's dynamic atlas texture to a custom `Font3DMaterial` **once**, when the text object is created:

```csharp
textMesh.font.RequestCharactersInTexture(textMesh.text);          // one-time glyph request
material.SetTexture("_MainTex", textMesh.font.material.mainTexture);  // one-time atlas binding
```

Unity's dynamic font atlas is small and never grows - when it overflows with glyph requests (which happens *constantly* with hanzi, and even during studio boot), Unity **replaces the texture with a fresh one**. Every already-created text object keeps referencing the orphaned old texture, while its mesh UVs are regenerated against the new atlas - so glyphs sample empty atlas regions and **silently vanish**. Which characters survive depends on rendering order, which is why the loss looks random and changes between runs. English text rarely triggers it (few dozen glyphs); Chinese/Japanese text triggers it constantly.

## The fix

- Keep a registry of every `TextMesh` the plugin creates
- Subscribe to `Font.textureRebuilt`
- On every atlas rebuild: re-bind the fresh atlas texture to all registered text objects and re-rasterize the glyphs of currently visible ones
- Log each rebuild with the new atlas size and the number of re-bound objects:

```
[Message:Studio Text Plugin] [AtlasFix] Dynamic font atlas rebuilt (#1). New atlas size: 256x256. Re-bound 3 text object(s), re-rasterized 2 visible one(s).
```

No more pre-playing the timeline hoping the atlas settles before recording - rebuilds become invisible in real time **and** during VideoExport renders.

## Building

```
dotnet build src/KK_StudioTextPlugin -p:GameDir="C:\Path\To\Koikatsu"
```

`GameDir` can also come from the `KK_DIR` environment variable. Requires BepInEx 5 installed in the game folder. The two compiled `.resources` blobs under `src/KK_StudioTextPlugin/Precompiled` are checked in (the resx contains binary file references that Core MSBuild's ResGen cannot process) and are regenerated with the `genres` recipe above their folder if you change the embedded assets.

## Installation

Replace `BepInEx/plugins/jim60105/KK_StudioTextPlugin.dll` (v20.08.05.0) with the built `KK_StudioTextPlugin.dll` (v20.08.06.1). Same plugin GUID `com.jim60105.kk.studiotextplugin`, so configs and scene cards are unaffected.

## Notes

- A `GlyphRasterSize` config (default `500` = original behavior) was added as an experimental knob; lowering it reduces the glyph rasterization size while compensating with `characterSize`, keeping the visual size identical. Real-world capacity effect depends on how the engine rasters - with the textureRebuilt fix in place, capacity no longer affects correctness.
- Based on [jim60105/KK](https://github.com/jim60105/KK) tag `v20.08.06.0` (last Koikatsu release, identical in-tree version 20.08.05.0 to the distributed dll).

## License

[CC BY-NC-SA 3.0 TW](./LICENSE.html) - same license as the upstream plugin (姓名標示-非商業性-相同方式分享 3.0 台灣). Not affiliated with Illusion.
