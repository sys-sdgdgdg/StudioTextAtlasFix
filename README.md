# StudioTextAtlasFix

> **Result: the atlas-rebind approach was a net negative and has been reverted.**
> The latest code here (v20.08.06.4) is a faithful, behavior-identical rebuild of
> jim60105's original plugin. For actual use, prefer the original distributed
> `KK_StudioTextPlugin` v20.08.05.0 - the experiment and its findings are kept
> below for the record.

## What this repo set out to fix

Koikatsu studio text plugin (`KK_StudioTextPlugin`, by [jim60105](https://github.com/jim60105/KK)):
CJK text objects **randomly lose characters** in studio scenes - some glyphs of a
sentence silently vanish, run to run differently.

## What was tried, and what the experiments showed

**Theory**: the plugin binds the font's dynamic atlas texture to its custom
`Font3DMaterial` only once. When Unity rebuilds the atlas (constant pressure
under CJK), the texture object is replaced and existing materials keep sampling
the orphaned old texture.

**Fix attempt (v20.08.06.1 - .2)**: register every created `TextMesh`, hook
`Font.textureRebuilt`, re-bind the fresh atlas texture on every rebuild.

**Empirical result**: text got **worse than the original** - whole sentences
dropped to one or two visible characters. A/B testing against the original
dll with the same scene card proved it:

1. **Unity 5.6 TextMesh meshes are frozen** - mesh UVs are generated once (at
   text/font/size set time) and are NOT regenerated when the font atlas rebuilds.
2. The original plugin's one-time texture binding is therefore load-bearing:
   **old UVs + old (orphaned) texture = a permanently valid pair**. Frozen
   materials also stop participating in atlas pressure, which lets the churn settle.
3. Re-binding materials to the new atlas **breaks the frozen pair** (old UVs +
   new texture = sampling empty atlas regions), actively destroying texts that
   would have kept rendering.

An additional `GlyphRasterSize` experiment (rasterize smaller, compensate with
`characterSize`) made text completely invisible - the assumed relationship
between `fontSize` and world size does not hold, and a leftover `250` value in
the config file kept breaking subsequent builds until cleaned up.

A deferred-rebind verification pass confirmed the rebind itself latched
correctly (0 stale materials) - the mechanism worked as designed; the design
was wrong about Unity's mesh behavior.

## Where this leaves the original bug

The original's random partial loss remains unsolved. The plausible direction
for a real fix, based on the findings:

- Per-text glyph baking: render each text object's glyphs once into a private
  texture (TextMeshPro-style static atlas per object), completely bypassing the
  shared dynamic atlas. Substantial work, not attempted here.

## Building

```
dotnet build src/KK_StudioTextPlugin -p:GameDir="C:\Path\To\Koikatsu"
```

`GameDir` can also come from the `KK_DIR` environment variable. Requires BepInEx 5
installed in the game folder. The two compiled `.resources` blobs under
`src/KK_StudioTextPlugin/Precompiled` are checked in (the resx contains binary file
references that Core MSBuild's ResGen cannot process) and are regenerated from the
original embedded assets.

## License

[CC BY-NC-SA 3.0 TW](./LICENSE.html) - same license as the upstream plugin (姓名標示-非商業性-相同方式分享 3.0 台灣). Not affiliated with Illusion.
