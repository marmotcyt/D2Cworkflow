# D2C Workflow

This workflow converts Figma design exports into HTML/CSS.

It works together with the Figma Night Heron export Plugin. The plugin reads the selected Figma frame and exports the structured design package. The D2C workflow then uses that package to recreate the design as a standalone `index.html` file.

Plugin link:
https://www.figma.com/community/plugin/1621368854448144560/nightheron-json-extractor
## How It Works

1. Select a frame or component in Figma.
2. Run the Figma Night Heron Plugin.
3. Download the exported ZIP package.
4. Unzip the package into a working folder.
5. Run the D2C workflow on the exported files.
6. Review the generated `index.html` against the reference image.

## Exported Package

The Figma Export Plugin creates:

```text
figma_export.zip
├── design.json
├── skeleton.txt
└── assets/
    ├── reference.png
    └── exported images and SVGs
```

- `design.json` contains the structured layout, styles, text, and asset references.
- `skeleton.txt` provides a simplified layer hierarchy for quick understanding.
- `assets/` contains the exported visual assets and the reference screenshot.

## Using It With the Figma Plugin

Use the Figma Export Plugin before starting the D2C workflow. The exported files are the input source for the conversion.

For best results:

- Export one clear frame or component at a time.
- Use the normal optimized export for regular D2C work.
- Use debug export when you need to inspect the full layer tree without deduplication.
- Treat the exported `design.json` as the source of truth for layout and styling.

## Output

The D2C workflow generates:

```text
index.html
```

The HTML file uses the exported JSON and assets to recreate the selected Figma design.

## Recommended Flow

```text
Figma Design
  -> Figma Export Plugin
  -> Exported ZIP Package
  -> D2C Workflow
  -> index.html
  -> Visual Review
```

