# freetype

[FreeType](https://freetype.org) 2 for [Avin](https://github.com/avin-lang/avin): font faces opened
from files and from memory, glyphs loaded and rendered, kerning, raw SFNT tables, and the axes and
named instances of variable fonts. The package binds FreeType's C library directly through Avin's C
bindings, which check every binding against FreeType's own headers when the program is built.

Needs Avin 0.0.20 or later.

## Installing FreeType

The package finds FreeType with `pkg-config` (the module `freetype2`):

- macOS: `brew install freetype`
- Debian and Ubuntu: `apt install libfreetype-dev`

## Using it

Name it in your package's `avin.hocon`:

```hocon
dependencies {
  freetype { git = "github.com/avin-lang/freetype", version = "0.1.0" }
}
```

and import from the module `dev.avinlang.freetype`.

## An example

The measures of a glyph at 48 pixels:

```avin
import dev.avinlang.freetype.{FreeTypeError, describe, library}

measure(path: string, c: char) -> Result[unit, FreeTypeError] =
    val lib = library()?
    val face = lib.openFace(path)?
    face.setPixelSizes(0, 48)?
    face.loadChar(c)?
    face.renderGlyph()?
    println(s"${face.familyName().getOrElse("?")}: ${face.bitmapWidth()} x ${face.bitmapRows()} pixels")
    println(s"advance ${face.advance().x / 64} pixels, top ${face.bitmapTop()} above the baseline")
    Ok(())

main(args: Vector[string])
    measure(args(0), 'あ') match
        Ok(_) -> ()
        Err(e) -> println(describe(e))
```

## What is bound

- **Libraries**: `library()` (`FT_Init_FreeType`) and a library's `version()`; `openFace(path, index)`
  and `openMemoryFace(block, index)` over an `avin.c.ByteBlock`, which the face keeps, since FreeType
  reads it for as long as the face lives.
- **Faces**: the counts, names and measures of `FT_FaceRec` (`faceCount`, `index`, `flags`,
  `styleFlags`, `glyphCount`, `familyName`, `styleName`, `fixedSizeCount`, `charmapCount`,
  `unitsPerEm`, `ascender`, `descender`, `height`, `maxAdvanceWidth`, `underlinePosition`,
  `underlineThickness`, `boundingBox`), the flag tests (`isScalable`, `isSfnt`, `isFixedWidth`,
  `hasKerning`, `hasMultipleMasters`, `hasGlyphNames`, `isItalic`, `isBold`,
  `namedInstanceCount`) and `postscriptName`.
- **Sizes**: `setCharSize` (points at a resolution), `setPixelSizes` and `sizeMetrics`.
- **Character maps**: `charIndex`, `codeIndex`, `nameIndex`, `firstChar`, `nextChar`, `characters`
  and `selectCharmap` with the `Encoding` mirror.
- **Glyphs**: `loadGlyph` and `loadChar` with `LoadFlags` (FreeType's `FT_LOAD_*`, combined with `|`),
  `renderGlyph` with the `RenderMode` mirror, and what the glyph slot holds: `glyphIndex`,
  `glyphMetrics`, `advance`, `linearHoriAdvance`, and the bitmap's `bitmapRows`, `bitmapWidth`,
  `bitmapPitch`, `bitmapGrays`, `bitmapLeft` and `bitmapTop`.
- **Kerning**: `kerning(left, right, mode)` answers an `Offset` (`FT_Vector`) in the `KerningMode`
  asked for.
- **SFNT tables**: `sfntTable(tag)` answers a table's bytes as the font holds them.
- **Variations**: `mmVar()` answers an `MMVar` with its `Axis` and `NamedStyle` values;
  `setDesignCoordinates`, `designCoordinates`, `setNamedInstance` and `defaultNamedInstance`.
- **Errors**: every call that can fail answers `Err(Failed(call, code))`, the code one of the
  `FT_Err_*` constants (`cannotOpenResource`, `unknownFileFormat`, `invalidArgument`, `tableMissing`,
  …); `describe` and `errorString` put it in words.

The glyph slot's bitmap is read three ways: `bitmapRows`, `bitmapWidth`, `bitmapPitch` and
`bitmapPixelMode` (a `PixelMode`, with `glyphFormat` a `GlyphFormat`); `bitmap()`, a view of
`|pitch| * rows` bytes that is valid only until the face is used again; and `renderChar` and
`renderIndex`, which answer a `Glyph` whose pixels are copied out, top row first, whatever the sign
of the pitch. `boundingBox`, `sizeMetrics`, `glyphMetrics` and `advance` answer FreeType's own structs
(`FT_BBox`, `FT_Size_Metrics`, `FT_Glyph_Metrics`, `FT_Vector`) read whole, in the units FreeType keeps
them in (26.6 pixels for a size's and a glyph's measures); `charmapEncoding(i)` and `charmapEncodings()`
answer each character map's `Encoding`.

Libraries, faces and variation descriptors are handles on FreeType's counted objects, given back when
they are collected or at once by `close()`. A face holds a reference on its library and a descriptor
one on its face, so either may outlive the handles it was made from.

## Testing

```
avin test
```

The tests open the bundled fonts in `tests/fonts`, each with its SIL Open Font License: Kaisei Decol
Regular, PT Sans (for its `kern` table) and Quicksand (variable in weight).
