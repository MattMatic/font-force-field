# font-force-field: Font Force Field Tool
Web based glyph collision and distance tool for complex OpenType fonts.

This tool is **completely offline** (i.e. nothing is sent to the cloud).

Having used Simon Cozen's excellent `collidoscope` project, but also been
badly bitten by Python library version nightmares, this was created as an
off-the-shelf tool.

While developing this tool, various other features became possible - like GID based
overrides for distances.

# Live
https://mattmatic.github.io/font-force-field

# Usage

- Drag and drop a font file onto the Font table cell (it will turn yellow)
    - Note: font name and version is shown
- Adhoc test:
  - Enter a word to show the glyphs, collisions, and other data
  - Use the sliders to set the force field sizes
- JSON definitions
  - You can define a JSON file for your optimum settings for the font under test
  - The JSON file can also include glyph ID overrides for special use cases
- Word list processing:
  - Drag and drop a .TXT file onto the Word List area on the right
  - (Recommend you use a JSON file to define the settings)
  - Click "Rate!" to iterate through all words and produce a list
  - When done, the "Save List" will download a text file of words and the rating outputs.
  - You can slo click "Font Compare Word List" to transfer everything over to the word list processing tool!

# Rating output abbreviations
The rating output that is used in the list:
- `GG#` Collision Glyph-to-Glyph. i.e. the glyphs genuinely overlap
- `cb#` Collision Base-to-Base Near Field
- `cx#` Collision Mark-to-Base Near Field
  - Either Near Field to Near Field
  - Or Near Field to Near-Base Field (depending on the relationship between the mark glyph and the base glyph)
- `cm#` Collision Mark-to-Mark Near Field
- `ca:####` Collision Area value. The `####` value is in design-units-squared
- `ok` Far Force Field might be ok
  - Where there is a gap in the Far Fields for the base, but the Far Fields of the marks join it all together
- `fb` Far force field shows a gap for the base glyphs
- `fb##` Extra far force field shows a gap. e.g. `fb50` = 150%
- `mg##` Extra far force field with marks glued in shows a gap. e.g. `mg25` = 125%
- `fm` Far force field shows a gap for the mark glyphs

HINT: if the output text _starts with_ `ok` then there are no collisions, and the word _might_ be ok if there is no `fm` in the output.

# JSON file definition

Suggestions:
- It is recommended you specify the font name and version.
- The `base` amd `mark` values **must** be defined as absolute (font design units)
- The GID overrides can use either percentages of the `base` or `mark` values, or can override in font design units
- Abbreviations:
  - `n` = Near Force Field
  - `nb` = Near Force Field for Mark-to-Base (or mark to cursive group)
  - `f` = Far Force Field

```json
{
  font:{
    name:'MyFontName',
    version:'Exact Version',
    //glyphNames: true,     // if we are referring to Glyph Names in the definitions
  },  // Warning if different

  // 'n'  = Near
  // 'f'  = Far
  // 'nb' = Near Base (mark to glyph or cursive group)
  // 'cl' = 1 means allowed to overlap with cluster set

  base:{n:45, f:95},

  mark:{n:10, nb:5, f:100},

  // Optional overrides of far fields measurements for rating
  // The more values, the longer the rating process takes!
  far:{
    values:[10,25,50],  // (default) => 110%, 125%, 150%
    nearPath: 'near',   // (default is undefined). Choose the field for marks-glued to far field (e.g. 'near', 'glyph', 'far')
    ok: true,           // (default is false). True will scan for ok percentage (takes even longer)
  },

  // GID specific (use quotes around numbers)
  '643':{n:'120%',f:'200%'},  // Using percentages
  '644':{n:15,    f:50},      // Explicit values

  '454':{n:'80%', nb:'50%', f:40},  // Mixed. `nb` = mark-to-base near field

  // Named groups of GIDs for use with `bar` and `overlaps`
  groups: [
    consonants: [102,103,105,'110-125'], // Including a range (use quotes!)
    marks: [203,405,205],
  ],

  // For Indic scripts with joining top bar
  bar : [
    {
      // The area(s) can be either horizontal bars, or rectangles.
      // Several areas can be defined.
      // Only GIDs in this set remove the 'area' from collision checks
      area: [
        [600, 650],               // exclusion between y=600 and y=650
        [50, 300, 80, 400],       // Rectangle (50, 300) - (80, 400)
      ],
      gids: [56, 58, 70, 100],  // GIDs that are in the 'bar' set
      far: false,                 // false=bar is not in the far, true=bar included in far
    },
  ],

  // Array of arrays for GIDs that are allowed to overlap
  // Useful for Indic scripts with non-cursive glyphs that overlap in areas other than the 'bar' area
  overlaps: [                   
    [56, 59, 61],                 // i.e. 56 cannot overlap 57
    [57, 63, 75, 85],
    [consonants, 75, 85],         // The `group` called `consonant` and 75 and 85
  ],

  defaults: {
    cl: true,                    // Says that attached marks are allowed to overlap within the cluster by default (without this it does not allow unless the GID overrides)
  },

  '123' : {cl:true},              // Override cluster overlap for one GID
}
```

Ranges and group references are also supported in most fields:
```json
…
  '100-104,115,123-126': {nb: -50},
  'marks,300-305' : {nb: -100},
…
```

# Glyph Names
You can use either GIDs or glyph names in the definitions for glyph and/or sets.
By default, the wildcard matching is similar to DOS filename matching.

Definition:
- `START` - begins with
- `MATCH$` - full match
- `*END`  - ends with
- `START*END` - starts and ends with
- `START*MID*` - starts with and contains
- `START*MID*END` - starts with, contains, and ends with.
- `/REGEXP/` - regular expression

Examples:
- `TE` will match all derivatives, e.g. "TEm16", "TEi3" etc
- `TEm` will match all "TEm16", "TEm2", but not "TEi3" etc
- `TEm1` will only match "TEm1" and will not match "TEm16"
- `*Em*`` will find "BEm1", "TEm2", etc
- `*.one` will find "HAMZA_ABOVE.one" etc


_Suggest you add to the `font` section the line `glyphNames: true,` to ensure the JSON will verify there are names in the font file._


> If a string match no glyphs, you will see either an error `Error: Group/Range error "GLYPHNOTTHERE"` or `TypeError: Cannot read properties of null (reading 'GLYPHNOTTHERE')` (where 'GLYPHNOTTHERE' is the string you hoped was a glyph name)
> 
> (Use the "Copy Glyph Names" to copy and then paste into a text editor to see the names available.)

# JavaScript Debug Functions
There are some useful functions to help diagnose issues (use the Developer Console).

- `list.gids(TXT)` - list all the glyph IDs, where TXT is a range, a group name, or glyph name prefix (or regular expression).
- `list.groups()` - show all group names
- `list.group(NAME)` - show the source text and glyph IDs for the specified group NAME.
- `list.overlaps(GID)` or `list.overlaps()` - show the overlaps that GID is part of (or show all overlaps). Shows the index, the source text, and the glyph IDs.
- `list.bar()` - show the glyph IDs for the bar feature.

> NOTE: the Debug dialog shows more comprehensive outputs.

# Technical Details
Please see: https://github.com/MattMatic/font-force-field/blob/main/TECHNICAL.md
