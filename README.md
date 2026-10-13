# MASAMI

> An experimental visual perception engine built in C — somewhere between machine vision, live surveillance imagery, low-level systems programming and emotional glitch aesthetics.

---

## Overview

`blobtracker` is an experimental computer vision and image processing system written in C.

It started as a low-level experiment in blob detection and visual manipulation. It has since evolved into a modular perception pipeline capable of processing static images and analyzing live camera frames, measuring detected regions, rendering visual annotations and exporting transformed images.

What began as a small image-processing experiment is becoming something stranger:

- A low-level computer vision engine
- A live camera perception system
- A geometric region analysis pipeline
- A procedural image annotation tool
- A foundation for experimental visual rendering
- An exploration of how machine perception can become an artistic language

The project operates directly on pixel data rather than relying on a high-level computer vision framework. Image decoding, pixel traversal, neighborhood exploration, component measurements and visual overlays are implemented through a modular C codebase.

The current system supports two execution modes:

- **Image mode:** Process one or multiple images and export annotated results.
- **Camera mode:** Capture live frames through FFmpeg, analyze them through the perception pipeline and display the results through SDL2.

The goal is not simply to recognize objects.

It is to construct a machine that observes, measures and manifests its interpretation of visual information.

And to see what happens when that interpretation is imperfect.

---

## Philosophy

Most computer vision systems are designed to produce increasingly accurate interpretations of the world.

`blobtracker` begins somewhere more primitive: pixels, brightness, adjacency, geometry and a machine attempting to make sense of what it sees.

Its current perception model is deliberately simple. It identifies connected regions of sufficiently bright pixels, measures their geometry and highlights selected candidates.

There is no semantic understanding of what those regions represent. A component is not inherently a person, an animal or an object. It is a region that satisfies the engine's current rules.

That limitation is also an interesting starting point.

What does a machine see when its entire interpretation of reality is built from numerical thresholds and spatial relationships?

What happens when those interpretations become the material for visual expression?

Here, technical experiments and aesthetic intent are not separate concerns.

The system exists at the intersection of:

- Computer vision and image processing
- Systems programming and memory-level computation
- Live visual input and surveillance-inspired interfaces
- Geometric analysis and spatial measurements
- Procedural rendering and glitch aesthetics
- Experimental perception and transfem cyber-aesthetics

The engine does not need to understand everything it sees.

It needs to make its perception visible.

---

## System Architecture

The project evolved from a monolithic implementation into a modular processing pipeline.

Its modules separate input acquisition, image analysis, component detection, visualization and output generation.

### Execution overview

```text
                         BLOBTRACKER
                              |
                         main.c / CLI
                              |
                 +------------+------------+
                 |                         |
              -img                       -cam
                 |                         |
          Image Acquisition          Camera Observer
           image_devouring()          observer()
                 |                         |
             stb_image              FFmpeg / V4L2
                 |                         |
                 +------------+------------+
                              |
                       RGBA PIXEL DATA
                              |
                       image_dissecting()
                              |
                       nebula_detection()
                              |
                        object_watcher()
                              |
                   Component Measurements
                              |
                     Component Filtering
                              |
                     Rank Largest Regions
                              |
                    object_containment()
                              |
                 +------------+------------+
                 |                         |
             -img OUTPUT               -cam VIEW
                 |                         |
           essence_enacted()         SDL2 Rendering
                 |                         |
            *_renewed.jpg             Live Display
```

The two execution modes share the same underlying perception pipeline. They differ in how visual data enters the system and how the resulting frames are presented.

---

## Execution Modes

### 01 — Static Image Processing

The image-processing mode accepts multiple image paths in a single invocation.

Each image is decoded into a four-channel pixel buffer, passed through the detection pipeline, annotated with the selected component bounding boxes and exported as a new image.

The original input is not intentionally overwritten.

```bash
./Masami -img image.png
```

Multiple images can be submitted together:

```bash
./Masami -img image.png portrait.jpg landscape.png
```

Each image is processed independently.

The output filename preserves the input basename and adds the `_renewed` suffix.

```text
INPUT                         OUTPUT
------------------------------------------------
image.png                     image_renewed.jpg
portrait.jpg                  portrait_renewed.jpg
landscape.png                 landscape_renewed.jpg
```

The exported image contains the transformations produced by the current pipeline, including the magenta bounding-box overlays.

### 02 — Live Camera Perception

The camera mode introduces a continuous visual input path.

Instead of reading a file once, the system receives camera frames through an FFmpeg subprocess, processes each frame and displays the annotated result using SDL2.

```bash
./Masami -cam 640 480
```

The current implementation:

1. Opens the Linux V4L2 camera device at `/dev/video0`.
2. Requests the specified frame dimensions.
3. Uses FFmpeg to produce raw RGBA frames through a pipe.
4. Passes each frame through the existing perception pipeline.
5. Updates an SDL2 streaming texture with the processed pixels.
6. Presents the frame in a live window.
7. Emits diagnostic and telemetry messages to the terminal.

The current window identifies itself as:

```text
[MASAMI] - GLaring
```

The observer logs frame counts and the number of displayed detections every 30 processed frames.

The window can be closed normally or stopped using `ESC`.

Camera acquisition currently depends on Linux V4L2, FFmpeg and SDL2. It is an early live-processing loop, not yet a complete video analytics platform.

---

## The Perception Pipeline

The current engine performs brightness-based connected-region detection.

It does not use a trained neural network, a semantic classifier or an external object detection framework.

Instead, it builds its interpretation from raw pixel values and spatial relationships.

### 01 — Image acquisition

`image_devouring()`

Loads an image from disk and initializes the data structures required for processing.

Responsibilities include:

- Loading image pixels through `stb_image`
- Normalizing the processing buffer to four channels
- Initializing the image and perception state
- Invoking the processing pipeline
- Exporting the resulting image
- Releasing the loaded image buffer

The corresponding live-camera path initializes the same image representation using incoming RGBA frames.

### 02 — Pipeline orchestration

`image_dissecting()`

Coordinates the analysis of a frame.

It allocates a visitation map, initializes the working measurements, invokes the detection stage, orders accepted components by size and draws bounding boxes around selected candidates.

The visitation map ensures that pixels already examined during region traversal are not repeatedly treated as new starting points.

### 03 — Brightness analysis

`nebula_detection()`

Scans the image pixel by pixel and calculates a simple mean of the red, green and blue channels.

The current detection threshold is hard-coded:

```c
bright = (red + green + blue) / 3;

if (bright > 110 && !eye->visited[pixel_index])
{
    object_watcher(img, eye, x, y);
}
```

Pixels above the threshold become candidate starting points for component exploration.

The threshold is currently fixed in the implementation; runtime configuration is not yet available.

### 04 — Connected-component exploration

`object_watcher()`

Explores neighboring pixels to identify a connected bright region.

Despite the original recursive-perception concept, the current implementation uses an **iterative, stack-based traversal**, avoiding recursive function calls for this operation.

The detector examines an eight-directional neighborhood:

```text
  NW   N   NE
    \  |  /
  W -- P -- E
    /  |  \
  SW   S   SE
```

During traversal, the engine:

- Marks discovered pixels as visited
- Explores neighboring pixels above the brightness threshold
- Accumulates pixel coordinates
- Counts region pixels
- Tracks minimum and maximum coordinates
- Constructs a bounding rectangle around each candidate

The traversal uses a bounded working stack and checks image boundaries before accessing neighboring pixels.

### 05 — Geometric measurements

Each candidate region is evaluated using several measurements.

| Metric | Meaning |
|---|---|
| Centroid | Approximate center calculated from the mean pixel coordinates |
| Bounding box | Minimum rectangle enclosing the region |
| Width and height | Dimensions of the bounding rectangle |
| Pixel count | Number of pixels included in the detected component |
| Density | Component pixel count relative to bounding-box area |
| Aspect ratio | Width-to-height relationship |

These values form the basis for filtering candidates rather than treating every bright region as a meaningful detection.

### 06 — Candidate filtering

Not every connected region becomes an accepted blob.

The current implementation applies geometric and size constraints, including:

- Minimum bounding-box width and height of 10 pixels
- More than 250 pixels in a candidate region
- Component density greater than `0.20`
- An initial aspect-ratio constraint that excludes extremely elongated regions
- A maximum storage capacity of 512 accepted components

These thresholds are experimental parameters rather than universal definitions of an object.

They are intended to reduce noise and discard small or unusually shaped regions that are less useful for the current visual output.

### 07 — Ranking and visualization

After detection, accepted components are sorted by pixel count in descending order.

The current pipeline selects up to eight of the largest regions for annotation.

`object_containment()` draws a rectangular outline around each selected component, using the engine's characteristic magenta color:

```text
RGB: 191, 0, 255
```

The overlay is drawn directly into the image buffer.

The result is a visual representation of the engine's interpretation of the frame: not a semantic label, but a geometric statement about what it considers worth observing.

---

## Core Structures

The perception pipeline uses three central data structures.

### Soulcatcher

The container for the image being processed.

```c
typedef struct s_soulcatcher
{
    int width;
    int height;
    int channels;
    unsigned char *data;
} Soulcatcher;
```

It holds the dimensions and pixel buffer used by the image-processing modules.

### Theeye

The working state of the perception engine.

```c
typedef struct s_theeye
{
    int b_sum_x;
    int b_sum_y;
    int b_pixels;
    unsigned char *visited;
    Blob blobs[512];
    int blob_count;
} Theeye;
```

It centralizes temporary traversal state, the visitation map, accumulated measurements and the collection of accepted components.

The current implementation initializes this state for each image or camera frame. It does not yet maintain persistent identities for detections across frames.

### Blob

The representation of a detected visual component.

```c
typedef struct
{
    int id;
    int center_x;
    int center_y;
    int min_x;
    int min_y;
    int max_x;
    int max_y;
    int obj_index;
    int b_pixels;
    int width;
    int height;
    float density;
    float aspect_ratio;
} Blob;
```

The structure carries identification within the current detection result, spatial measurements and geometric statistics.

These measurements provide a foundation for more advanced tracking and perception systems.

---

## Current Codebase

The project is organized into separate modules rather than concentrating the engine inside a single source file.

```text
blobtracker/
|
+-- main.c
|   `-- Command-line interface and execution modes
|
+-- includes/
|   `-- Shared headers and image library headers
|
+-- io/
|   +-- stb_invoke.c
|   |   `-- stb_image implementation definitions
|   |
|   +-- bottle.c
|   |   `-- Image acquisition and artifact export
|   |
|   +-- lens.c
|   |   `-- Camera input, FFmpeg pipe and SDL2 display
|   |
|   +-- processing.c
|       `-- Detection, geometric analysis and overlays
|
+-- Makefile
    `-- Build configuration
```

### Module responsibilities

| Module | Responsibility |
|---|---|
| `main.c` | Parses command-line arguments and selects the execution mode |
| `io/stb_invoke.c` | Defines the image library implementations in one translation unit |
| `io/bottle.c` | Loads static images and exports processed results |
| `io/lens.c` | Captures live camera frames and displays the running pipeline |
| `io/processing.c` | Detects connected regions, computes measurements, filters candidates and draws overlays |
| `includes/core.h` | Defines the shared structures and function interfaces |
| `Makefile` | Compiles and links the modular executable |

The current Makefile produces an executable named `Masami`, while the repository retains the `blobtracker` name. The live interface and terminal diagnostics also use the `MASAMI` identity.

---

## Current Capabilities

The current implementation includes:

- Modular C architecture
- Static image input through `stb_image`
- Multiple image paths in one invocation
- RGBA image processing
- Linux V4L2 camera capture through FFmpeg
- Live frame presentation through SDL2
- Pixel-wise brightness analysis
- Visitation maps
- Eight-directional connected-region traversal
- Bounded stack-based exploration
- Component size and shape measurements
- Centroid estimation
- Density and aspect-ratio filtering
- Ranking of accepted regions by pixel count
- Magenta bounding-box rendering
- Automatic output naming
- JPEG artifact export
- Frame-count and detection-count telemetry

These capabilities form the current foundation of the engine. They do not imply semantic object recognition, persistent multi-object tracking or complete video-file processing.

---

## Roadmap

### Perception and tracking

- Persistent component identities between frames
- Temporal object memory
- Multi-object tracking
- Motion persistence
- Optical-flow experimentation
- Configurable detection thresholds
- More flexible component
