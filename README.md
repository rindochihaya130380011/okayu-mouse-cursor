from PIL import Image
import os, shutil, json
frames_dir="/mnt/data/frames"

# generate preview gif
frames=[]
files=sorted(os.listdir(frames_dir))
for f in files:
    frames.append(Image.open(os.path.join(frames_dir,f)))
gif_path="/mnt/data/preview.gif"
frames[0].save(gif_path, save_all=True, append_images=frames[1:], loop=0, duration=100)

# README
readme="""# Okayu Animated Cursor Pack

This pack includes 64-frame animation cursor resources for macOS (Mousecape).

## Contents
- frames/ — individual PNG frames
- preview.gif — animation preview
- license.txt
- okayu.cape — basic Mousecape cape file

## Installation
1. Install Mousecape: https://github.com/alexzielenski/Mousecape
2. Open `okayu.cape` or import frames manually.
"""

readme_path="/mnt/data/README.md"
with open(readme_path,"w") as f: f.write(readme)

# license
lic="MIT License\n\nCopyright (c) 2025"
lic_path="/mnt/data/license.txt"
with open(lic_path,"w") as f: f.write(lic)

# create simple cape (plist)
cape={
    "HotSpots":[{"x":0,"y":0}],
    "Frames": files,
    "FrameDuration":0.1
}
import plistlib
cape_path="/mnt/data/okayu.cape"
with open(cape_path,"wb") as f:
    plistlib.dump(cape,f)

# assemble zip
pack="/mnt/data/okayu-cursor-pack"
os.makedirs(pack, exist_ok=True)
shutil.copy(readme_path, pack)
shutil.copy(lic_path, pack)
shutil.copy(gif_path, pack)
shutil.copy(cape_path, pack)
# copy frames
dst_frames=os.path.join(pack,"frames")
shutil.copytree(frames_dir, dst_frames, dirs_exist_ok=True)

zip_final="/mnt/data/okayu-cursor-pack.zip"
shutil.make_archive("/mnt/data/okayu-cursor-pack", 'zip', pack)

zip_final
