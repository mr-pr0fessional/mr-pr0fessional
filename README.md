"""
hologram.py — generates a movie-style (Iron Man HUD) 360-hologram GIF
from a portrait image, in ONE self-contained file.

Usage:
    pip install pillow numpy
    python hologram.py [input_image] [output.gif]

Defaults:
    input  : Screenshot_2026-09-28_232318.png   (next to this script)
    output : hologram-ironman.gif
"""
import math
import random
import sys
from pathlib import Path

import numpy as np
from PIL import Image, ImageDraw, ImageFilter

# ----------------------------------------------------------------------------
# Settings — tweak these freely
# ----------------------------------------------------------------------------
SRC = sys.argv[1] if len(sys.argv) > 1 else "Screenshot_2026-09-28_232318.png"
OUT = sys.argv[2] if len(sys.argv) > 2 else "hologram-ironman.gif"

CROP = (170, 0, 680, 793)   # (left, top, right, bottom) region of the portrait
CANVAS_W, CANVAS_H = 500, 600
FIG_W = 340                 # width of the floating hologram figure
FRAMES = 48                 # frames in the loop
FRAME_MS = 70               # ms per frame
BASE_Y = 556                # where the emitter disc sits
GIF_COLORS = 80

# Iron Man HUD palette (dark -> bright)
PALETTE = np.array([
    [2, 8, 18],       # background-level dark
    [0, 40, 70],      # deep blue
    [0, 110, 160],    # mid cyan
    [40, 200, 235],   # bright cyan
    [190, 245, 255],  # near-white glow
], dtype=np.float32)

random.seed(42)
np.random.seed(42)


# ----------------------------------------------------------------------------
# Helpers
# ----------------------------------------------------------------------------
def tint_cyan(img: Image.Image) -> Image.Image:
    """Map image luminance onto the cyan HUD palette."""
    lum = np.asarray(img.convert("L"), dtype=np.float32) / 255.0
    lum = lum ** 0.85  # lift midtones so the figure stays visible
    stops = np.linspace(0.0, 1.0, len(PALETTE))
    out = np.zeros((*lum.shape, 3), dtype=np.float32)
    for ch in range(3):
        out[..., ch] = np.interp(lum, stops, PALETTE[:, ch])
    return out


def add(base: np.ndarray, layer: np.ndarray) -> np.ndarray:
    """Additive (screen-like) blend used for every glow element."""
    return np.clip(base + layer, 0, 255)


def vertical_gradient(w: int, h: int, top_a: float, bottom_a: float,
                      color=(60, 200, 235)) -> np.ndarray:
    """Vertical alpha gradient rectangle, as an RGB float layer."""
    ramp = np.linspace(top_a, bottom_a, h, dtype=np.float32)[:, None]
    layer = np.zeros((h, w, 3), dtype=np.float32)
    for ch in range(3):
        layer[..., ch] = ramp * color[ch]
    return layer


# ----------------------------------------------------------------------------
# Build the static hologram "figure" layer once
# ----------------------------------------------------------------------------
src = Image.open(SRC).convert("RGB").crop(CROP)
scale = FIG_W / src.width
fig_h = int(src.height * scale)
figure = src.resize((FIG_W, fig_h), Image.LANCZOS)

# Soft alpha mask: dark background of the portrait vanishes, bright figure stays.
lum = np.asarray(figure.convert("L"), dtype=np.float32) / 255.0
alpha = np.clip((lum - 0.13) / 0.26, 0, 1) ** 1.2
alpha_img = Image.fromarray((alpha * 255).astype(np.uint8)).filter(
    ImageFilter.GaussianBlur(1.5))
alpha = np.asarray(alpha_img, dtype=np.float32)[..., None] / 255.0

holo_base = tint_cyan(figure) * alpha

# Bloom: blurred copy adds the movie-style glow bleed
bloom = np.asarray(
    Image.fromarray(np.clip(holo_base, 0, 255).astype(np.uint8)).filter(
        ImageFilter.GaussianBlur(7)),
    dtype=np.float32,
) * 0.75 * alpha
holo_base = add(holo_base, bloom)

# Static scanlines etched into the figure
scan = np.ones((fig_h, 1, 1), dtype=np.float32)
scan[::3, 0, 0] = 0.72
holo_base *= scan

FIG_X = (CANVAS_W - FIG_W) // 2
FIG_BOTTOM = BASE_Y - 6                 # figure floats just above the disc
FIG_TOP = FIG_BOTTOM - fig_h

# ----------------------------------------------------------------------------
# Particles rising from the emitter
# ----------------------------------------------------------------------------
N_PARTICLES = 30
parts = []
for _ in range(N_PARTICLES):
    parts.append({
        "x": random.uniform(60, CANVAS_W - 60),
        "y": random.uniform(120, BASE_Y),
        "r": random.uniform(0.8, 2.4),
        "speed": random.uniform(1.2, 3.4),
        "twinkle": random.uniform(0, 2 * math.pi),
    })

# ----------------------------------------------------------------------------
# Per-frame render
# ----------------------------------------------------------------------------
canvas_bg = np.zeros((CANVAS_H, CANVAS_W, 3), dtype=np.float32)
# subtle dark-blue radial-ish gradient backdrop
yy, xx = np.mgrid[0:CANVAS_H, 0:CANVAS_W].astype(np.float32)
center_dist = np.sqrt(((xx - CANVAS_W / 2) / (CANVAS_W / 2)) ** 2 +
                      ((yy - CANVAS_H / 2) / (CANVAS_H / 2)) ** 2)
bg_glow = np.clip(1.0 - center_dist, 0, 1) ** 2 * 14.0
for ch, c in enumerate((4, 16, 30)):
    canvas_bg[..., ch] = c + bg_glow * (c / 14.0)

# Emitter disc (drawn once): bright core + rings
disc_layer = np.zeros_like(canvas_bg)
disc_img = Image.fromarray(np.zeros((CANVAS_H, CANVAS_W, 3), dtype=np.uint8))
d = ImageDraw.Draw(disc_img)
cx = CANVAS_W // 2
for rx, ry, col in [
    (140, 27, (0, 60, 90)), (110, 20, (0, 110, 160)),
    (76, 13, (40, 200, 235)), (40, 7, (120, 230, 252)),
]:
    d.ellipse([cx - rx, BASE_Y - ry, cx + rx, BASE_Y + ry], fill=col)
# bright lens-flare line across the disc
d.line([cx - 175, BASE_Y, cx + 175, BASE_Y], fill=(90, 210, 240), width=2)
disc_layer = np.asarray(disc_img, dtype=np.float32)
disc_glow = np.asarray(
    Image.fromarray(disc_layer.astype(np.uint8)).filter(ImageFilter.GaussianBlur(9)),
    dtype=np.float32,
) * 0.9

# Light cone from the emitter up to the figure (drawn once, animated per frame)
cone_h = FIG_BOTTOM - 60
cone = vertical_gradient(CANVAS_W, cone_h, top_a=0.10, bottom_a=0.42,
                         color=(30, 170, 220))
cone_mask_img = Image.new("L", (CANVAS_W, cone_h), 0)
dm = ImageDraw.Draw(cone_mask_img)
dm.polygon([(cx - 170, cone_h), (cx + 170, cone_h),
            (cx + FIG_W // 2 - 20, 40), (cx - FIG_W // 2 + 20, 40)], fill=255)
cone_mask_img = cone_mask_img.filter(ImageFilter.GaussianBlur(14))
cone_mask = np.asarray(cone_mask_img, dtype=np.float32)[..., None] / 255.0
cone *= cone_mask
cone_full = np.zeros_like(canvas_bg)
cone_top = FIG_BOTTOM - cone_h
cone_full[cone_top:cone_top + cone_h] = cone

glitch_rows = sorted(random.sample(range(FRAMES), 7))
frames = []
for i in range(FRAMES):
    t = i / FRAMES
    frame = canvas_bg.copy()

    # --- projector flicker -------------------------------------------------
    flicker = 1.0 + 0.10 * math.sin(2 * math.pi * t * 6) + random.uniform(-0.06, 0.06)
    if i in glitch_rows and random.random() < 0.5:
        flicker *= 0.55          # projector stutter
    if random.random() < 0.05:
        flicker *= 1.25          # bright sparkle

    # --- light cone --------------------------------------------------------
    cone_frame = cone_full * (0.8 + 0.2 * math.sin(2 * math.pi * t * 3)) * flicker
    frame = add(frame, cone_frame * 0.55)

    # --- emitter -----------------------------------------------------------
    pulse = 0.85 + 0.15 * math.sin(2 * math.pi * t * 4)
    frame = add(frame, disc_glow * flicker * pulse)
    frame = add(frame, disc_layer * flicker * pulse)

    # --- figure: bob + jitter + glitch --------------------------------------
    bob = int(round(3.5 * math.sin(2 * math.pi * t * 2)))
    jitter = random.randint(-2, 2) if random.random() < 0.35 else 0
    holo = holo_base * flicker

    # rolling scanline band sweeping down the figure
    band_center = (t * 1.6 % 1.0) * fig_h
    band = np.exp(-((np.arange(fig_h) - band_center) / 26.0) ** 2)
    holo *= (1.0 + 0.55 * band)[:, None, None]

    # occasional horizontal slice displacement + chromatic split
    if i in glitch_rows:
        holo = holo.copy()
        for _ in range(random.randint(2, 4)):
            y0 = random.randint(0, max(1, fig_h - 40))
            hgt = random.randint(8, 36)
            shift = random.randint(-22, 22)
            holo[y0:y0 + hgt] = np.roll(holo[y0:y0 + hgt], shift, axis=1)
        holo[..., 0] = np.roll(holo[..., 0], 3, axis=1)   # red split
        holo[..., 2] = np.roll(holo[..., 2], -3, axis=1)  # cyan split

    # paste figure (max-blend keeps it glowing over the cone)
    x0 = FIG_X + jitter
    y0 = FIG_TOP + bob
    region = frame[y0:y0 + fig_h, x0:x0 + FIG_W]
    frame[y0:y0 + fig_h, x0:x0 + FIG_W] = np.maximum(region, holo)

    # --- particles ----------------------------------------------------------
    part_img = Image.fromarray(np.zeros((CANVAS_H, CANVAS_W, 3), dtype=np.uint8))
    dp = ImageDraw.Draw(part_img)
    for p in parts:
        py = (p["y"] - i * p["speed"]) % (BASE_Y - 90) + 90
        tw = 0.5 + 0.5 * math.sin(p["twinkle"] + i * 0.55)
        b = int(90 + 150 * tw)
        r = p["r"]
        dp.ellipse([p["x"] - r, py - r, p["x"] + r, py + r], fill=(int(b * 0.4), b, b))
    frame = add(frame, np.asarray(part_img, dtype=np.float32) * flicker)

    # --- vignette -----------------------------------------------------------
    frame *= (1.0 - 0.35 * np.clip(center_dist - 0.55, 0, 1))[..., None]

    img = Image.fromarray(frame.astype(np.uint8))
    frames.append(img.quantize(colors=GIF_COLORS, dither=Image.FLOYDSTEINBERG))

frames[0].save(OUT, disposal=2, save_all=True, append_images=frames[1:],
               duration=[FRAME_MS] * FRAMES, loop=0, optimize=True)
print(f"saved {OUT} — {len(frames)} frames, {CANVAS_W}x{CANVAS_H}")


