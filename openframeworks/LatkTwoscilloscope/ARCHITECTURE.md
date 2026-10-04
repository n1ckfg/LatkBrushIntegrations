# Architecture of LatkTwoscilloscope

LatkTwoscilloscope is an openFrameworks application that plays back a Latk animation through [ofxTwoscilloscope](https://github.com/n1ckfg/ofxTwoscilloscope), the way the addon's `example-transform` treats a vector shape: each frame is projected to the screen, encoded as one loop of XY oscilloscope audio, run through a chain of audio effects, and drawn back from the altered audio. The strokes are drawn by a simulated oscilloscope beam in their own colours, or decoded back into vector strokes. The altered loop also plays out of the sound card, so it can drive a real scope in X-Y mode.

## Directory Structure
- `src/`
  - `main.cpp` - Opens a 1024×768 window and runs `ofApp`.
  - `ofApp.h` & `ofApp.cpp` - Loads the drawing, steps the animation, sets up the effect chain and its panel, plays the audio, and handles input.
  - `LatkScopeRenderer.h` & `LatkScopeRenderer.cpp` - Projects and encodes the strokes, runs the round trip, and draws the result.
- `addons.make` - `ofxGui`, `ofxLatk`, `ofxPoco` and `ofxTwoscilloscope`.
- `bin/data/` - The `.latk` drawings.

## Core Components

### 1. Application Layer (`ofApp`)
- **Loading:** `Latk` reads the `.latk` archive directly.
- **Animation:** instead of `latk.run()`, which both advances and draws, `update()` advances each layer's frame at Latk's 12 fps and `draw()` does the drawing.
- **Camera:** the strokes are projected by hand and only drawn inside `cam.begin()` in the original-lines view, so `setup()` connects `ofEasyCam` to the mouse with `cam.setEvents()`, and `windowResized()` updates its mouse area. The camera ignores the mouse while the pointer is over the panel, so dragging a slider doesn't orbit it.
- **Effects:** the same chain as `example-transform`, in the same order and with the same defaults: a 1500 Hz low pass and a 0.6 ms Y delay on, the rest off. Every setting is in the `ofxGui` panel, along with the renderer's own (loop frequency, beam size and intensity).
- **Audio:** an `XYscope` loops the altered audio (X left, Y right) on the default output, and gets the new loop every frame.
- **Controls:** `l` cycles the view (beams, decoded strokes, original lines), `e` solos the next effect, `n` turns them all off, `m` mutes, `g` hides the panel, `s` saves the decoded strokes as SVG, `w` saves four seconds of the altered audio (X, Y, Z) as WAV, and `o` writes `test.latk`.

### 2. Rendering (`LatkScopeRenderer`)
`update()` runs the whole round trip every frame, as `example-transform` does. It takes about 1 ms for the jellyfish at the default 5 Hz loop.

1. **Project:** every point of the current frames goes through the camera's model-view-projection matrix, computed once. Strokes are cut where they leave the camera's depth range. They're also cut exactly at the window's edges, because the window is the scope's canvas and anything past it would clip the audio. Each visible run becomes a *piece* with its stroke's colour.
2. **Encode:** the pieces are written as one loop of XYscope-format audio: X, Y and Z, with the canvas mapped to -1..1 and +Y up. The loop is a whole number of samples (8,820 at 5 Hz). Each piece gets a blanked sample that jumps the beam to its start, then lit samples spaced evenly along it, from end to end. The lit samples are shared out by length, so the beam moves at an even speed as it does in XYscope's waveforms. Every piece gets at least two. If the loop is too short for that, the shortest pieces are left out.
3. **Transform:** `XYTransformer::transform()` runs the effects over five repeats of the loop, so filters and echoes settle, and keeps the last one.
4. **Draw:** depending on the view:
   - **Beams:** each piece's lit samples become an `OsciMesh`, the Oscilloscope app's beam renderer: a quad per pair of samples, lit by a gaussian beam sweeping along it and drawn additively. Pieces are grouped into one mesh per colour. The beam leaves less light the faster it moves. Brightness is scaled by the average step between samples, so strokes come out at about `beam intensity` however long the loop is and however far the camera is zoomed.
   - **Decoded strokes:** each piece's samples are decoded back into polylines with `XYDecoder::decodeCycle()`, and drawn in the piece's colour at Latk's line width.

The strokes are encoded here and not by `XYscope::buildWaves()`, because the app needs to know which samples belong to which stroke. XYscope spreads all its shapes over one wavetable and resamples it. A short stroke can share a sample with the next one, so counting blanks doesn't reliably find it. With the app's own encoding, every sample's stroke is known. The effects pass Z through untouched and keep samples in place, so that's still true after them. Whatever an effect does to a stroke's samples, the stroke keeps its colour.

## Data Flow Pipeline
1. **Load:** `ofxLatk` unzips and parses the `.latk` file into layers, frames and strokes of 3D points.
2. **Advance:** `ofApp::update()` steps each layer's current frame.
3. **Project:** `LatkScopeRenderer` projects the current frames to the window and cuts them into visible pieces.
4. **Encode:** the pieces become one loop of X, Y, Z audio, with every sample tagged with its piece.
5. **Transform:** `XYTransformer` runs the loop through the effect chain.
6. **Output:** the altered loop is drawn as coloured beams or decoded strokes, and played by `XYscope`.

## Dependencies
- **openFrameworks:** windowing, camera, sound stream, math and OpenGL wrappers.
- **ofxLatk:** reads and writes Latk data, and draws the original GL lines.
- **ofxTwoscilloscope:** the effect chain, `XYTransformer`, `XYDecoder`, the `OsciMesh` beam renderer, and `XYscope` for playback and WAV rendering.
- **ofxGui:** the settings panel.
- **ofxPoco:** listed from the original example; this app doesn't use it, since `ofxLatk` now unzips `.latk` files itself.
