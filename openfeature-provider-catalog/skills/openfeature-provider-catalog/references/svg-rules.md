# SVG Rules

Use the repository's existing AGENTS.md instructions for general editing and validation workflow.

New provider logo files belong in `static/img/` and use the `*-no-fill.svg` naming convention.

## Allowed

- `<svg>` metadata such as `xmlns` and `viewBox`.
- Geometry elements such as `<path>`, `<rect>`, `<circle>`, `<polygon>`, and `<g>`.
- Geometry winding attributes such as `fill-rule` and `clip-rule`.
- `currentColor` only when the repository's existing convention explicitly requires it, although geometry-only paths without paint attributes are preferred for new provider logos.

## Forbidden for new provider logos

- `fill`, `stroke`, `color`, or `style` attributes.
- `fill-opacity`, `stroke-opacity`, `opacity`, gradients, masks, filters, or paint servers.
- Hard-coded colors, including hex, RGB, HSL, named colors, or CSS variables.
- Embedded raster images or `<image>` elements.

The logo should render as one theme-controlled color supplied by the consuming component. Preserve the vendor's recognizable silhouette rather than approximating it with unrelated shapes.

## Check

Run this from the repository root, replacing the path as needed:

```sh
if rg -n 'fill=|stroke=|color=|style=|paint-|opacity=|<image|#[0-9A-Fa-f]{3,8}|rgb\\(|hsl\\(' static/img/vendor-no-fill.svg; then
  echo 'Forbidden SVG paint declaration found'
  exit 1
fi
```

Also inspect the rendered shape when the source logo is complex. Bitmap tracing tools can help, but clean vendor-supplied vector geometry or an exact manually provided path is preferred.
