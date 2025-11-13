okayu-animated-cursor/
├── build_pack.py         # main Python build script (split frames, gif, plist, zip)
├── make_release.sh       # helper shell script to build zip + git release steps
├── README.md             # user-facing repo README
├── license.txt           # MIT license
├── .gitignore
└── assets/
    └── spritesheet.png   # (<img width="1024" height="1024" alt="okayucursor 2_MM_24_4_8_5_F_1 5_WW_ 2" src="https://github.com/user-attachments/assets/7fc42e5d-e0f6-49bf-b0e9-51ef4872e970" />)
#!/usr/bin/env python3
"""
build_pack.py
Build an Okayu animated cursor pack from a spritesheet.

Requirements:
    pip install pillow

Usage:
    python3 build_pack.py \
        --input assets/spritesheet.png \
        --out-dir dist \
        --name okayu \
        --cols 8 --rows 8 \
        --frame-duration 100

Output:
    dist/
      ├── okayu-cursor-pack/         # assembled folder
      │    ├── frames/               # individual numbered frames
      │    ├── preview.gif
      │    ├── okayu.cape
      │    ├── README.md
      │    └── license.txt
      └── okayu-cursor-pack.zip
"""
import os
import argparse
from PIL import Image
import shutil
import plistlib
import textwrap

def split_spritesheet(path, cols, rows, out_dir):
    img = Image.open(path).convert("RGBA")
    w, h = img.size
    cw = w // cols
    ch = h // rows
    os.makedirs(out_dir, exist_ok=True)
    frames = []
    total = cols * rows
    for r in range(rows):
        for c in range(cols):
            box = (c * cw, r * ch, (c + 1) * cw, (r + 1) * ch)
            frame = img.crop(box)
            fname = f"frame_{r*cols+c:02d}.png"
            fpath = os.path.join(out_dir, fname)
            frame.save(fpath)
            frames.append(fname)
    return frames

def make_preview_gif(frames_dir, frames_list, out_path, duration_ms=100):
    images = [Image.open(os.path.join(frames_dir, f)).convert("RGBA") for f in frames_list]
    # Save first with append_images
    images[0].save(out_path, save_all=True, append_images=images[1:],
                   loop=0, duration=duration_ms, disposal=2)

def make_cape(frames_list, out_path, hotspot=(0,0), duration=0.1):
    # Minimal .cape plist structure. Mousecape uses a plist format.
    cape = {
        "HotSpots": [{"x": hotspot[0], "y": hotspot[1]}],
        "Frames": frames_list,
        "FrameDuration": duration,
        "Version": 1
    }
    with open(out_path, "wb") as f:
        plistlib.dump(cape, f)

def write_readme(out_path, pack_name):
    content = f"""# {pack_name} Animated Cursor Pack

This pack contains an animated cursor exported from a spritesheet, packaged for macOS (Mousecape).

## Contents
- frames/ — PNG frames (ordered)
- preview.gif — animation preview
- {pack_name}.cape — simple Mousecape plist file
- README.md
- license.txt

## Installation (macOS)
1. Install Mousecape: https://github.com/alexzielenski/Mousecape
2. Download and unzip the pack.
3. Option A — Double click `{pack_name}.cape` and Apply in Mousecape.
4. Option B — Import frames manually in Mousecape and set Frame Duration to 0.07–0.12s.

## Notes
- Hotspot used in the .cape is at the top-left by default. Adjust inside Mousecape if needed.
"""
    with open(out_path, "w") as f:
        f.write(content)

def copy_license(src_license, dst):
    shutil.copyfile(src_license, dst)

def build(args):
    spritesheet = args.input
    cols = args.cols
    rows = args.rows
    out_dir = os.path.abspath(args.out_dir)
    pack_folder_name = f"{args.name}-cursor-pack"
    pack_path = os.path.join(out_dir, pack_folder_name)
    frames_path = os.path.join(pack_path, "frames")

    # Clean
    if os.path.exists(pack_path):
        shutil.rmtree(pack_path)
    os.makedirs(frames_path, exist_ok=True)

    print(f"[+] Splitting {spritesheet} into frames ({cols}x{rows})...")
    frames = split_spritesheet(spritesheet, cols, rows, frames_path)

    print("[+] Creating preview GIF...")
    gif_path = os.path.join(pack_path, "preview.gif")
    make_preview_gif(frames_path, frames, gif_path, duration_ms=args.frame_duration)

    print("[+] Writing .cape file...")
    cape_path = os.path.join(pack_path, f"{args.name}.cape")
    # .cape references frames by filenames (not full paths)
    make_cape(frames, cape_path, hotspot=(args.hotspot_x, args.hotspot_y), duration=args.frame_duration/1000.0)

    print("[+] Writing README and copying license...")
    write_readme(os.path.join(pack_path, "README.md"), args.name)
    if args.license:
        copy_license(args.license, os.path.join(pack_path, "license.txt"))
    else:
        # fallback: write minimal MIT
        with open(os.path.join(pack_path, "license.txt"), "w") as f:
            f.write("MIT License\n\nCopyright (c) 2025")

    # Zip it
    zip_out = os.path.join(out_dir, f"{pack_folder_name}.zip")
    print(f"[+] Zipping into {zip_out} ...")
    shutil.make_archive(os.path.join(out_dir, pack_folder_name), 'zip', pack_path)

    print("[+] Build complete.")
    print("Pack folder:", pack_path)
    print("Zip file:", zip_out)

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", "-i", required=True, help="Input spritesheet image")
    parser.add_argument("--out-dir", "-o", default="dist", help="Output folder")
    parser.add_argument("--name", "-n", default="okayu", help="Pack name")
    parser.add_argument("--cols", type=int, default=8, help="Spritesheet columns")
    parser.add_argument("--rows", type=int, default=8, help="Spritesheet rows")
    parser.add_argument("--frame-duration", type=int, default=100, help="Frame duration in ms for preview GIF")
    parser.add_argument("--hotspot-x", type=int, default=0, help="Hotspot X coordinate")
    parser.add_argument("--hotspot-y", type=int, default=0, help="Hotspot Y coordinate")
    parser.add_argument("--license", type=str, default=None, help="Path to license file to copy into pack")
    args = parser.parse_args()
    build(args)
#!/usr/bin/env bash
set -e

# Usage: ./make_release.sh assets/spritesheet.png
SPRITESHEET=${1:-assets/spritesheet.png}
NAME=${2:-okayu}
OUTDIR=${3:-dist}

# Create dist and run build
python3 build_pack.py -i "$SPRITESHEET" -o "$OUTDIR" -n "$NAME" --cols 8 --rows 8 --frame-duration 100

echo "Build finished. Zip is in ${OUTDIR}/${NAME}-cursor-pack.zip"

# Optional: quick git steps (uncomment to use)
# git add .
# git commit -m "Add ${NAME} cursor pack"
# git tag -a v1.0 -m "v1.0"
# git push origin main --tags
# Okayu Animated Cursor Pack

This repository contains tools and assets to build an animated cursor pack for macOS (Mousecape).

## Build
Requirements:
- Python 3.8+
- pillow (`pip install pillow`)

```bash
# Example
python3 build_pack.py --input assets/spritesheet.png --out-dir dist --name okayu --cols 8 --rows 8 --frame-duration 100
