---
layout: post
title: "Maya Asset Manager Plug-in"
date: 2026-02-27
categories: [Maya, Plugin]
image: "post-img/maya-asset-manager-plugin/Cover.png"
tags: [Maya, Python, Plugin]
published: true
---

## Why Need

During my personal project, I found myself accumulating a growing number of Maya assets — models, HDR environments, Alembic caches — scattered across different folders on disk. Every time I wanted to reuse something, I had to dig through File Explorer, open the file just to check what was inside, and then import it manually.

Maya's built-in Content Browser does help, but its preview is limited to a fixed set of image and scene formats, with no support for HDR, EXR, or Alembic files. It also has no tag system — search only works by filename within the current folder.

So I decided to build my own asset manager — one with thumbnail previews for all the formats I actually use, a tag system for quick filtering, and a one-click publish workflow that captures a viewport screenshot and stores everything in a clean folder structure.

---

## Demo Video

{% include embed/youtube.html id='GA2W4SiAW_E' %}
[Maya Asset Manager - QiRen](https://youtu.be/GA2W4SiAW_E)

---

## UI Design

The interface is built with PySide6 and adopts a frameless dark theme to blend naturally with Maya's native UI style.

The layout is divided into three panels. The left panel is a folder tree navigator that reflects the actual directory structure of the registered asset libraries, allowing quick browsing by category. The centre panel displays assets as a thumbnail grid, with the item count shown at the bottom and a size slider for adjusting thumbnail scale. A search bar in the top-right supports filtering by asset name or tag. The right panel serves as an inspector, showing metadata for the selected asset including name, format, file size, date, renderer, path, and tags, alongside Capture and Upload buttons for setting the asset thumbnail.

The Publish button in the top bar opens the export dialog, which provides options for format, proxy, texture, and thumbnail settings before committing the asset to the library.

---

## Data Structure Design

The data structure needs to serve two purposes: persist asset information on disk so it survives between sessions, and provide fast in-memory access during browsing.

Config File: A single JSON file at ~/.maya_asset_manager_config.json stores the global settings — registered library paths and appearance preferences. This means the plugin remembers your libraries across Maya sessions without touching the project files.

```python
{
    "libraries": [
        "D:/Assets/Models",
        "D:/Assets/HDR"
    ],
    "appearance": {
        "font_family": "Microsoft YaHei",
        "font_size": 12,
        "show_folder_icons": true
    }
}
```

Per-Asset Metadata: Each asset has a sidecar _meta.json file stored alongside it on disk. This keeps the metadata co-located with the asset, so moving or copying a folder preserves all its information.

```python
{
    "display_name": "Fountain",
    "tags": ["prop", "outdoor", "stone"],
    "renderer": "redshift",
    "icon_path": null
}
```

In-Memory Asset Dict: When scanning, each asset is loaded into a dictionary and stored in a list. The UI reads directly from this list, and clicking an item retrieves its data by key:

```python
asset = {
    "name": "Fountain",
    "original_name": "Fountain",
    "path": "D:/Assets/Models/Fountain/Fountain.ma",
    "thumbnail": "D:/Assets/Models/Fountain/Fountain.jpg",
    "tags": ["prop", "outdoor"],
    "renderer": "redshift"
}
```

This flat dictionary approach keeps the scanning and display logic simple — the folder tree in the UI reflects the actual directory structure on disk, so reorganising assets is just a matter of moving folders and rescanning.

---

## Implementation Details

### 1. Recursive Asset Scanning with Thumbnail Resolution

The plugin scans all registered library directories recursively, supporting formats including .ma, .mb, .fbx, .abc, .usd, .hdr, and .exr. For each asset, it attempts to resolve a thumbnail by checking several naming patterns before falling back to a utility search. HDR and EXR files without an existing preview will trigger automatic thumbnail generation.

```python
def _scan_directory_recursive(self, directory):
    assets = []
    valid_exts = [".ma", ".mb", ".hdr", ".exr", ".obj", ".fbx", ".abc", ".usd", ".usda", ".usdc"]
    
    for root, dirs, files in os.walk(directory):
        dirs[:] = [d for d in dirs if d.lower() != "textures"]
        for file in files:
            ext = os.path.splitext(file)[1].lower()
            if ext in valid_exts:
                # Thumbnail resolution
                preview_candidates = [
                    f"{name}_preview.jpg", f"{name}_preview.png",
                    f"{name}.jpg", f"{name}.png"
                ]
                for cand in preview_candidates:
                    if os.path.exists(os.path.join(root, cand)):
                        thumb = os.path.join(root, cand)
                        break

                # Auto-generate for HDR/EXR
                if not thumb and ext in [".hdr", ".exr"]:
                    gen_thumb_path = os.path.join(root, f"{name}_preview.jpg")
                    if thumb_gen.generate_thumbnail(full_path, gen_thumb_path):
                        thumb = gen_thumb_path
```

### 2. Tag System with Chip UI

Assets can be tagged and browsed through a dedicated Tag Manager window. All tags across registered libraries are aggregated into a single view, displayed as styled chips. Clicking a tag filters and emits the associated asset list to the main panel. The system also handles migration from the legacy .txt tag format to the current JSON metadata approach.

```python
def refresh_tags(self):
    all_tags = {}
    for lib in self.core.libraries:
        lib_tags = self.core.get_all_tags_in_root(lib)
        for tag, assets in lib_tags.items():
            if tag not in all_tags:
                all_tags[tag] = []
            all_tags[tag].extend(assets)

    for tag in sorted(all_tags.keys()):
        count = len(all_tags[tag])
        lbl = QtWidgets.QLabel(f"{tag} ({count})")
        lbl.setStyleSheet("""
            background-color: #3a3a3a; color: #ddd;
            padding: 4px 8px; border-radius: 4px;
            font-size: 11px; font-weight: bold;
        """)
```

### 3. Multi-Format Export Pipeline

A single export operation can produce multiple output formats simultaneously: Maya ASCII (.ma), Redshift Proxy (.rs), Arnold Standin (.ass), Alembic Cache (.abc), and GPU Cache. The renderer metadata field determines which proxy format is prioritised. A viewport screenshot is captured automatically as the asset thumbnail unless a custom image is provided.

```python
def export_asset(core, name, directory, selection_only=True,
                 thumbnail_mode="screenshot", export_proxy=False,
                 export_textures=False, export_alembic=False, export_gpu=False):

    # Maya ASCII
    cmds.file(filepath, force=True, type="mayaAscii", es=True)

    # Redshift Proxy
    if export_proxy and "redshift" in renderer.lower():
        cmds.file(proxy_path, force=True, type="Redshift Proxy", es=True)

    # Arnold Standin
    elif export_proxy and "arnold" in renderer.lower():
        cmds.arnoldExportAss(f=proxy_path, s=True, mask=63)

    # Alembic
    if export_alembic:
        cmds.AbcExport(j=f"-frameRange {start} {end} -uvWrite -file \"{abc_path}\"")

    # GPU Cache
    if export_gpu:
        cmds.gpuCache(startTime=start, endTime=end, dataFormat='ogawa',
                      directory=gpu_dir, fileName=gpu_file)
```

### 4. Texture Copy & Relink

Before exporting, the plugin copies all textures referenced by the selected objects into a Textures/ subdirectory and relinks the Maya file and Arnold image nodes to the new paths. After the export completes, all node paths are restored to their original values, leaving the working scene unaffected.

```python
# Pre-export: copy and relink
original_texture_paths = texture_copier.copy_textures(tex_dir, nodes=nodes_to_process)

try:
    cmds.file(filepath, force=True, type="mayaAscii", es=True)
finally:
    # Always restore original paths
    for node, path in original_texture_paths.items():
        if cmds.objExists(node):
            attr = "filename" if cmds.nodeType(node) == "aiImage" else "fileTextureName"
            cmds.setAttr(f"{node}.{attr}", path, type="string")
```
