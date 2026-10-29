# AI Air Scribble

Paint in the air with your index finger. A translucent canvas floats
directly on top of your live webcam feed — you can always see yourself
and your room right through your own painting, like drawing on glass.
Pick a color from the panel in the top-right corner, pick a brush tool
(freehand scribbling, or a perfect circle / rectangle / square / triangle
/ line / arrow), and paint.

**The main point of this app is free-form painting** — your exact
fingertip path, in the color you chose, translucent over the camera.
Perfect shapes only ever appear because you deliberately picked that
tool, the same way picking the rectangle tool in any paint program works
— nothing is auto-corrected unless you ask for it.

```
ai_air_scribble/
├── main.py               # webcam loop, translucent compositing, UI
├── hand_tracking.py      # MediaPipe hand-landmark wrapper
├── gesture_control.py    # turns landmarks into draw/select/pause/clear/undo/redo
├── shape_recognition.py  # explicit shape tools + optional auto-detector
├── drawing_canvas.py     # strokes, undo/redo, translucent layer compositing, save
├── config.py             # every tunable constant lives here
├── requirements.txt
└── README.md
```

## 1. Requirements

- **Python 3.9 – 3.11** (MediaPipe does not yet publish wheels for every
  newer Python version — if `pip install mediapipe` fails, this is almost
  always why; see Troubleshooting).
- A webcam.
- Libraries: `opencv-python`, `mediapipe`, `numpy` (see `requirements.txt`).

## 2. Installation

```bash
# 1. (recommended) create a virtual environment
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 2. install dependencies
pip install -r requirements.txt
```

## 3. Running it

```bash
python main.py
```

A single window opens: your live webcam feed fills it, with your
painting blended translucently on top, a color panel floating in the
top-right corner, and a toolbar along the bottom. Press **q** or **Esc**
at any time to quit (or use the on-screen **Exit** button).

## 4. Controls

### Hand gestures (air, no touching anything)
| Gesture | Action |
|---|---|
| ☝️ Index finger only | **Paint** — the fingertip is your brush |
| ✌️ Index + middle finger | **Finish stroke now** — ends the current stroke immediately |
| 🖐️ Open palm (all 4 fingers) | **Pause** — brush up, nothing is painted |
| ✊ Closed fist, **held ~1.2s** | **Clear** the canvas (the hold requirement stops an accidental fist from wiping your painting) |
| 👍 Thumb up alone | **Undo** |
| 👍🤙 Thumb + pinky up | **Redo** |

A stroke also finishes automatically if you hold the fingertip nearly
still for about 0.8 seconds while still painting — handy for slowly and
carefully drawn shapes.

### Mouse / on-screen UI
- **Top-right color panel** — click a swatch to pick **red, green, blue,
  yellow, white, or black**.
- **Bottom toolbar**:
  - **Clear / Undo / Redo / Save / Exit**
  - **Eraser** — toggles eraser mode; painting with it active removes
    paint (reveals the camera underneath) instead of adding color
  - **Brush Size** — cycles through 5 brush thicknesses
  - **Tool: ...** — cycles through the drawing tools (see below); click
    repeatedly, or press `t`, to step through them

### Keyboard shortcuts
`c` clear · `u` undo · `r` redo · `s` save · `e` toggle eraser ·
`t` cycle drawing tool · `1`–`5` pick a brush size · `v` toggle the hand
skeleton overlay · `q` / `Esc` quit

## 5. Drawing tools

| Tool | What you get |
|---|---|
| **Freehand** *(default)* | Your exact fingertip path, painted as-is — this is the main mode, pure air scribbling |
| **Circle** | The stroke you draw is replaced with the smallest circle that encloses it |
| **Rectangle** | Replaced with the stroke's bounding box |
| **Square** | Same as Rectangle, but forced to equal width and height |
| **Triangle** | A clean triangle built from the stroke's bounding box |
| **Line** | A straight line from where your stroke started to where it ended |
| **Arrow** | Same as Line, with an arrowhead at the end |
| **Auto-Detect** | *(optional)* the app guesses what shape you meant and only cleans it up if confident — otherwise your freehand stroke is kept |

Every tool except **Auto-Detect** is deterministic and immediate: a shape
only comes out "perfect" because you picked that tool on purpose, the
same way choosing the rectangle tool in any paint program works. There's
no misclassification risk because nothing is being guessed.

## 6. How the translucent canvas works

Every frame, `drawing_canvas.py` replays your whole painting — every
stroke, in order — onto two blank layers: one holding color, one holding
opacity (0–255) at each pixel. `main.py` then blends those layers on top
of the live camera frame: painted pixels are mixed with the camera image
underneath (default 78% paint, 22% camera), and every untouched pixel
shows the camera completely unmodified. That's what makes the canvas feel
like a pane of translucent glass sitting over your webcam feed rather
than a separate opaque panel.

Erasing works the same way: an eraser stroke sets the opacity layer back
to 0 wherever it passes. Because the whole painting is replayed in
stroke order every frame, undo/redo is just "replay one fewer/one more
stroke" — including eraser strokes, which undo and redo exactly like any
other stroke.

Saving (`Save` button, or `s`) exports the painting alone — not the
camera frame — as a **transparent PNG**, using that same opacity layer as
the file's real alpha channel, so you get your artwork on its own,
droppable onto anything afterward.

## 7. How hand tracking works

`hand_tracking.py` wraps Google's **MediaPipe Hands** solution, which runs
a small neural network on each webcam frame and returns 21 (x, y, z)
landmark points for a detected hand — one for each knuckle, fingertip,
and the wrist. We only need two things from that:

1. **The index fingertip position** (landmark #8) — used directly as the
   brush cursor.
2. **Which fingers are extended** — for every finger, we compare the
   fingertip landmark's height to the knuckle just below it (its "PIP"
   joint); if the tip is higher on screen than the knuckle, the finger
   counts as "up". The thumb moves sideways rather than up/down, so it's
   checked by comparing x-positions instead, and the direction flips
   depending on whether MediaPipe reports your hand as Left or Right.

That five-element up/down list (thumb, index, middle, ring, pinky) is all
`gesture_control.py` needs to recognize which gesture you're making.

## 8. How the explicit shape tools work

Each shape tool (except Auto-Detect) is a small, deterministic function
in `shape_recognition.py` that turns your raw stroke into geometry:

- **Circle** — `cv2.minEnclosingCircle()` on all the stroke's points.
- **Rectangle** — the stroke's own bounding box (min/max x and y).
- **Square** — the bounding box, with the larger of width/height used for
  both sides.
- **Triangle** — an isoceles triangle built from the bounding box: flat
  base along the bottom, apex centered on top.
- **Line / Arrow** — a straight segment from the stroke's first point to
  its last.

None of these need to measure confidence or guess your intent — you
already told the app what shape you wanted by picking the tool.

**Auto-Detect** (optional) is the one classifier left in the app: it
smooths the stroke, measures its circularity (`4·π·Area / Perimeter²`) to
separate circles from cornered shapes, and simplifies the outline with
`cv2.approxPolyDP()` across a range of simplification strengths to count
corners for triangle vs. rectangle/square. It only replaces your stroke
if it's at least `SHAPE_CONFIDENCE_THRESHOLD` (0.6 by default) confident;
otherwise your freehand stroke is kept exactly as drawn.

## 9. Troubleshooting

**"could not open the webcam" / a black window**
- Make sure no other application (Zoom, another browser tab, etc.) is
  currently using the camera.
- Try a different camera index: in `main.py`, change
  `cv2.VideoCapture(0)` to `cv2.VideoCapture(1)` (or 2, 3…).
- On Linux, confirm your user has permission to access `/dev/video0`
  (`sudo usermod -a -G video $USER`, then log out/in).
- On macOS, grant camera access to your terminal app in
  System Settings → Privacy & Security → Camera.

**`pip install mediapipe` fails**
- MediaPipe publishes wheels for a limited set of Python versions and
  platforms. Check `python --version` — 3.9–3.11 has the broadest
  support. If you're on a very new Python release, install 3.11 in a
  virtual environment instead.
- On Apple Silicon (M1/M2/M3), make sure you're running a native
  arm64 Python (not Python running under Rosetta).

**Hand detected but gestures feel unreliable**
- Keep your whole hand inside the frame and reasonably well lit —
  MediaPipe's landmark model struggles with strong backlighting or a hand
  that's partly cut off at the frame's edge.
- Hold each pose for a few frames; extremely fast gesture changes can be
  missed by the detector itself, before gesture logic ever sees them.

**The cursor lags behind my finger**
- Increase `CURSOR_SMOOTHING_ALPHA` in `config.py` (closer to 1.0 =
  less smoothing, less lag, but shakier). Default is `0.45`.

**Paint looks too solid / too faint against the camera**
- Adjust `CANVAS_OPACITY` in `config.py` (0.0 = invisible, 1.0 = fully
  solid, hiding the camera under every stroke). Default is `0.78`.

**Low frame rate**
- Lower `CAM_WIDTH` / `CAM_HEIGHT` in `config.py`.
- Close other applications competing for the webcam or CPU.
- Set `MAX_HANDS = 1` in `config.py` (already the default) — tracking a
  second hand roughly doubles MediaPipe's per-frame cost.
