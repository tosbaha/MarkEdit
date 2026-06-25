# Merging Upstream Changes into Our Fork

This document describes the process to pull upstream changes from
`MarkEdit-app/MarkEdit` into our fork while preserving our custom features.

## Quick Start

```bash
# In MarkEdit (main app)
git remote add upstream https://github.com/MarkEdit-app/MarkEdit.git
git fetch upstream main
git merge upstream/main --no-edit
git push origin main

# In MarkEdit-preview (extension)
git remote add upstream https://github.com/MarkEdit-app/MarkEdit-preview.git
git fetch upstream main
git merge upstream/main --no-edit
# Resolve any conflicts (see below)
git push origin main
```

## Conflict Resolution

### MarkEdit (main app)
Should merge cleanly. Our PDF export commit is on top of upstream `main`,
so `git merge upstream/main` applies upstream changes below ours.
No conflicts expected.

### MarkEdit-preview (extension)
Conflicts may occur in `dist/*.js` files (built artifacts).

**Resolution:**
- `dist/lite/markedit-preview.js` — accept upstream version
  (`git checkout --theirs dist/lite/markedit-preview.js`)
- `dist/markedit-preview.js` — accept our version
  (`git checkout --ours dist/markedit-preview.js`)

After resolving, rebuild the dist files with:
```bash
yarn install
yarn run build
```

## Verifying Our Changes Are Preserved

After merge, check that our PDF export files still exist:

### MarkEdit
```bash
git ls-files | grep -i pdf
# Should show:
#   CoreEditor/src/api/pdf.ts
#   CoreEditor/src/bridge/native/pdf.ts
#   MarkEditKit/Sources/Bridge/Native/Generated/NativeModulePDF.swift
#   MarkEditKit/Sources/Bridge/Native/Modules/EditorModulePDF.swift
```

### MarkEdit-preview
```bash
git diff upstream/main..HEAD --stat | grep -E "render|view|strings|main.ts"
# Should show PDF-related changes in:
#   main.ts
#   src/render.ts
#   src/view.ts
#   src/shared/strings.ts
```

## Our Custom Features

### 1. PDF Export from Preview Extension
- **Bridge module**: Native PDF generation via `EditorModulePDF.swift`
  + auto-generated `NativeModulePDF.swift`
- **API**: `MarkEdit.generatePDF(html, fileName)` exposed globally
- **Extension integration**: "Export as PDF" menu item in preview
- **Offline Mermaid**: Diagrams pre-rendered to inline SVGs before
  sending HTML to native — no CDN dependency
- **Light mode**: PDF always uses light color scheme regardless of
  system appearance

### Key Files

| File | Purpose |
|------|---------|
| `CoreEditor/src/bridge/native/pdf.ts` | TS bridge interface |
| `CoreEditor/src/api/pdf.ts` | `generatePDF()` API |
| `MarkEditKit/.../EditorModulePDF.swift` | Native module delegate |
| `MarkEditKit/.../NativeModulePDF.swift` | Auto-generated bridge code |
| `EditorViewController+Delegate.swift` | PDF rendering logic |
| `MarkEdit-preview/src/render.ts` | `applyStyles()` with color scheme param |
| `MarkEdit-preview/src/view.ts` | `exportPdf()`, `generateStaticHtml()` |
| `MarkEdit-preview/main.ts` | Menu item registration |
| `MarkEdit-preview/src/shared/strings.ts` | Localized strings |

## Build & Deploy

```bash
xcodebuild -project MarkEdit.xcodeproj \
  -scheme MarkEditMac \
  -configuration Release \
  -derivedDataPath /tmp/MarkEditRelease \
  clean build

rm -rf /Applications/MarkEdit.app
cp -R /tmp/MarkEditRelease/Build/Products/Release/MarkEdit.app /Applications/
```
