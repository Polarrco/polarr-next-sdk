# Polarr Next SDK

Polarr Next SDK implements a highly modular framework for manupulating RAW and bitmap data in the browser.

## Stack

It is built upon the following technologies:

- WebGL2: main rendering pipeline.
- TensorFlow: AI feature extraction and grouping, subject and sky matting.
- WASM: fast underlying libraries for IO and additional features.
    - RawSpeed: fast decoding & decompression of RAW files.
    - Exiv2: image metadata retrival (EXIF, XMP, etc).
    - Lensfun: vendor-specific lens correction information.

## Installation

The core module can be installed with the following command:

`npm install @polarr-next`

## Quick Start

The basic functionality on processing some RAW file and getting PNG file as a result of processing can be achieved with the following steps:

Install the necessary packages first:

```bash
npm install @polarr-next @polarr-next/io-raw @polarr-next/io-png @polarr-next/adjustments
```

After installation is done, the following code will import a RAW bytes, add some basic adjustments to it and then export PNG bytes:

```typescript
import { Renderer } from "@polarr-next"
import { RawIOAdapter } from "@polarr-next/io-raw"
import { PNGIOAdapter } from "@polarr-next/png-raw"
import { AdjustmentsPipeline } from "@polarr-next/adjustments"

// Create an offscreen renderer with a managed context
const renderer = new Renderer()

// Register all modules
renderer.use(RawIOAdapter)
renderer.use(PNGIOAdapter)
renderer.use(AdjustmentsPipeline) 

// Obtain (or create) File with one of supported RAW formats (for example, .arw file)
const response = await fetch("https://example.com/files/sample.arw")
const blob = await response.blob()
const rawFile = new File([blob], "sample.arw", { type: blob.type })

// Import the RAW file bytes into the pipeline
await renderer.import(rawFile)

// Set some basic adjustments 
renderer.setAdjustments({
    exposure: 0.12,
    saturation: -0.1,
    whites: 0.1,
    blacks: -0.1
})

// When ready, export data as PNG bytes
const outputPNGBlob = await renderer.export({
    format: "png"
})

// Download, save, or send the blob whenever 
```

This basic example illustrates the usage of Polarr Next framework modules to perform image processing. The code above can be executed in the worker, thus it will not block the main thread. 
 
## Introduction

*The core package* `@polarr-next` implements barebones importing and exporting capabilities. It also provides an interface to use existing modules or create new modules by implementing methods.

The core package is capable of the following features:

- Instantiate offscreen renderer with a managed WebGL context
    - On the main thread
    - In the worker
- Import image data bytes into the pipeline
- Export image data bytes from the pipeline
- Register new pipeline steps
- Registe import and export handlers

>`@polarr-next` does not itself do any rendering, but instead acts as a glue for all Polarr Next modules: it allows modules to connect to the pipeline and use the provided WebGL2 context to do image processing.

_Note that the framework currently only supports editing one image at a time._

### Importing Image Data

Importing data into the `Renderer` instance is possible with the asycnhronous `import` function:

```typescript
// Import the whole file if there is one
// Files can be artifically created
await renderer.import(file)
```

Or importing blob directly:

```typescript
try {
    // Import the whole file if there is one
    // Files can be artifically created
    await renderer.import({ type : "arw", data: someArrayBuffer })
}
catch (error) {
    console.log("File cannot be opened: it might not be supported:", error)
}
```

It is possible to monitor import progress and update UI accordingly when a certain resolution is available.

```typescript
try {
    // Import the whole file if there is one
    // Files can be artifically created
    await renderer.import({ 
        file,
        resolutions: ["half", "full"],
        onResolutionAvailable: (resolution) => {
            switch (resolution) {

                case "half":
                    console.log("Preview is available! Let's update the UI.")

                case "full":
                    console.log("Full resolution is available for precise zoomed-in editing.")

            }
        }
    })
}
catch {
    console.log("File cannot be opened: it might not be supported")
}
```

Certain IO modules provide granular updates because of the underlying complexity. For example, RAW import consists of several stages, and also of rendering the source images multiple times (firstly preview, then the full resolution image). All these state changes can be tracked with the `onResolutionAvailable` callback. 

### Exporting Image Data

Exporting currently imported image is made by calling the asynchronous `export` method on the `Renderer` instance:

```typescript
try {
    // Import the whole file if there is one
    // Files can be artifically created
    const jpegBlob = await renderer.export({
        format: "jpeg",
        onUpdate: (value) => {
            console.log("Exporting progress: ", value)
        }
    })
}
catch (error) {
    console.log("Export failed:", error)
}
```

Certain IO modules provide format-specific parameters that can be passed into the `export` method. 

It is possible to asycnhronously monitor the exporting progress by providing an optional `onUpdate` callback which will be called with a progress value in range `[0, 1]`.

## Import RAW 

`npm install @polarr-next/io-raw`

```typescript
import { RawIOAdapter } from "@polarr-next/io-raw"

renderer.use(RawIOAdapter)
```

Allows to import RAW files of formats supported by the underlying Rawspeed library:

```
dng, cr2, cr3, crw, pef, nef, arw, rw2, raw, raf, orf
```

```typescript
const file = await getSomeRAWFile()

await renderer.import({ 
    file,
    resolutions: ["half", "full"]
})
```

RAW module always provides a `half` resolution as soon as possible and then keeps loading `full` resolution if requested. The `full` resolution takes more time to be available from the underlying Rawspeed library, thus it is important to provide the `half` version first to update the UI.

### Convert RAW Into JPEG File

```typescript
import { Renderer } from "@polarr-next"
import { RawIOAdapter } from "@polarr-next/io-raw"
import { JPEGIOAdapter } from "@polarr-next/png-raw"

// Create an offscreen renderer with a managed context
const renderer = new Renderer()

// Register all modules
renderer.use(RawIOAdapter)
renderer.use(JPEGIOAdapter)

// Obtain (or create) File with one of supported RAW formats (for example, .arw file)
const response = await fetch("https://example.com/files/sample.arw")
const blob = await response.blob()
const rawFile = new File([blob], "sample.arw", { type: blob.type })

// Import the RAW file bytes into the pipeline
await renderer.import(rawFile)

// When ready, export data as PNG bytes
const outputJPEGBlob = await renderer.export({
    format: "jpeg",
    quality: 0.9
})

// Download, save, or send the blob whenever
```

## Import And Export PNG

`npm install @polarr-next/io-png`

```typescript
import { PNGIOAdapter } from "@polarr-next/io-png"

renderer.use(PNGIOAdapter)
```

Allows to import and export PNG files and process them similarly to how other formats are processed.

Import:

```typescript
const file = await getSomePNGFile()

await renderer.import({ 
    file
})
```

Export:

```typescript
// Export with PNG and PNG-specific parameters
await renderer.export({
    format: "png",
    colorspace: "srgb"
})
```

The PNG IO module does not support `half` resolution to be requested.

## Import And Export JPEG 

`npm install @polarr-next/io-jpeg`

```typescript
import { JPEGIOAdapter } from "@polarr-next/io-jpeg"

renderer.use(JPEGIOAdapter)
```

Allows to import and export JPEG files and process them similarly to how other formats are processed.

Import:

```typescript
const file = await getSomeJPEGFile()

await renderer.import({ 
    file
})
```

Export:

```typescript
// Export with JPEG and JPEG-specific parameters
await renderer.export({
    format: "jpeg",
    quality: 0.9,
    colorspace: "srgb"
})
```

The JPEG IO module does not support `half` resolution to be requested.

## Global And Local Adjustments 

`npm install @polarr-next/adjustments`

Provides set of adjustments that can be applied to the whole image or to the part of it.

### Global Adjustments

Provides basic set of adjustments to be applied to the whole loaded image region:

```typescript
export interface PointColorHSL {
	id: string
	color: [number, number, number]
	hueShift: number
	satShift: number
	lumShift: number
	range: number
	hueRange: [number, number, number, number]
	satRange: [number, number, number, number]
	lumRange: [number, number, number, number]
}

export interface Adjustments {
	// Global
	temperature: number
	tint: number
	exposure: number
	shadows: number
	highlights: number
	whites: number
	blacks: number
	contrast: number
	saturation: number
	vibrance: number
	vignette: number
	vignetteFeather: number
	texture: number
	clarity: number
	dehaze: number
    	sharpen: number

	// Point color
	pointColors: PointColorHSL[]

    	// Point curves
	curvesAll: Curve
	curvesRed: Curve
	curvesGreen: Curve
	curvesBlue: Curve

	// HSL
	hueRed: number
	saturationRed: number
	luminanceRed: number
	hueOrange: number
	saturationOrange: number
	luminanceOrange: number
	hueYellow: number
	saturationYellow: number
	luminanceYellow: number
	hueGreen: number
	saturationGreen: number
	luminanceGreen: number
	hueAqua: number
	saturationAqua: number
	luminanceAqua: number
	hueBlue: number
	saturationBlue: number
	luminanceBlue: number
	huePurple: number
	saturationPurple: number
	luminancePurple: number
	hueMagenta: number
	saturationMagenta: number
	luminanceMagenta: number

    	// Calibration
	calibrationRedHue: number
	calibrationRedSat: number
	calibrationGreenHue: number
	calibrationGreenSat: number
	calibrationBlueHue: number
	calibrationBlueSat: number

	// Parametric curves
	parametricDarksStop: number
	parametricMidsStop: number
	parametricLightsStop: number
	parametricHighlights: number
	parametricLights: number
	parametricDarks: number
	parametricShadows: number

	// Color grading
	gradingShadowsHue: number
	gradingShadowsSaturation: number
	gradingShadowsLuminance: number
	gradingMidtonesHue: number
	gradingMidtonesSaturation: number
	gradingMidtonesLuminance: number
	gradingHighlightsHue: number
	gradingHighlightsSaturation: number
	gradingHighlightsLuminance: number
	gradingGlobalHue: number
	gradingGlobalSaturation: number
	gradingGlobalLuminance: number
	gradingBlending: number
	gradingBalance: number

    	// Crop
	cropStraighten: number
	crop: Crop
	cropRatio: CropRatio

	// Rotation and flip
	orientation: number
	flipX: boolean
	flipY: boolean

    	// Transform & Perspective
	lensDistortion: number
	perspectiveVertical: number
	perspectiveHorizontal: number
	perspectiveRotate: number
	perspectiveScale: number
	perspectiveAspect: number
	perspectiveOffsetX: number
	perspectiveOffsetY: number

   	 // Grain
	grainAmount: number
	grainSize: number
	grainRoughness: number

   	 // Any number of local adjustments
    	localAdjustments: LocalAdjustmentsGroup[]
}
```

Any of the aforementioned global adjustments can be just passed into the `setAdjustments` function, which accepts `Partial<Adjustments>` type:

```typescript
renderer.setAdjustments({
    // Tinker highlights and shadows
    highlights: -0.12,
    shadows: 0.1

    // Add some grain
    grainAmount: 0.3,
    grainSize: 0.2,
    grainRoughness: 0.3,
})
```

Most adjustments are specified in range `[-1, 1]`, but some of them are in range `[0, 1]`, please refer to our [Adjustments Guide]() for more information about using global adjustments.

### Local Adjustments

Not all adjustments can be applied locally (to the part of the image). The following structure defines all possible adjustments that can be applied locally:

```typescript
export interface LocalAdjustments {
	temperature: number
	tint: number
	saturation: number
	toningHue: number
	toningSaturation: number
	exposure: number
	contrast: number
	highlights: number
	shadows: number
	whites: number
	blacks: number
	clarity: number
	texture: number
	dehaze: number
	sharpen: number
	pointColors: PointColorHSL[]
	strength: number
}
```

A set of local adjustments form `LocalAdjustmentsGroup`, which defines a set of masks that are joined together using the `ADD` and `SUBTRACT` operations:

```typescript
export interface LocalAdjustmentsGroup extends LocalAdjustments {
	active: boolean
	masks: Mask[]
}
```

The `masks` field specifies the region of interest on the image as a certain shape or an object. Its base is defined as following:

```typescript
export type MaskBlendMode = "add" | "subtract"

export interface Mask {
	active: boolean
	inverted: boolean
	blendMode: MaskBlendMode
}
```

Masks can be mixed together with `ADD` and `SUBTRACT` blending mode. 

Two masks are provided by this module:

Radial Mask:

```typescript
export interface RadialMask extends Mask {
	type: "radial"
	centerX: number
	centerY: number
	radiusX: number
	radiusY: number
	angle: number
	feather: number
}
```

Gradient Mask:

```typescript
export interface GradientMask extends BaseMask {
	type: "gradient"
	startX: number
	startY: number
	endX: number
	endY: number
}
```

Local adjustments can be applied to an image similarly to how global adjustments are applied:

```typescript
renderer.setAdjustments({
    // Global adjustment
    exposure: -0.1,

    // Local adjustments
    localAdjustments: [
        // The whole group consists of one mask
        {
            // Group information
            active: true,
            masks: [
                // One slightly feathered radial mask
                {
                    blendMode: "add",
                    active: true,
                    inverted: false,
                    type: "radial",
                    centerX: 0.5,
                    centerY: 0.5,
                    radiusX: 0.1,
                    radiusY: 0.2,
                    angle: 0,
                    feather: 0.2
                }
            ],
            // Group adjustments
            exposure: 0.2,
            contrast: 0.11
        }
    ]
})
```

Several masks can be combined together within one group to achieve precise image processing results.

## Denoise

`npm install @polarr-next/denoise`

This module depends on the adjustments module `@polarr-next/adjustments`

Incorporates advanced denoise into the processing pipeline. The denoise is then automatically used for all imported photos.

This module adds the following properties to be available via the `setAdjustments` method:

```typescript
export interface DenoiseAdjustments {
  luminanceNoiseReduction: number
	colorNoiseReduction: number
}
```

Usage:

```typescript
import { Renderer } from "@polarr-next"
import { RawIOAdapter } from "@polarr-next/io-raw"
import { JPEGIOAdapter } from "@polarr-next/png-raw"
import { AdjustmentsPipeline } from "@polarr-next/adjustments"
import { DenoisePipeline } from "@polarr-next/denoise"

// Create an offscreen renderer with a managed context
const renderer = new Renderer()

// Register all modules
renderer.use(RawIOAdapter)
renderer.use(JPEGIOAdapter)
renderer.use(AdjustmentsPipeline) 
renderer.use(DenoisePipeline) 

// Obtain (or create) File with one of supported RAW formats (for example, .arw file)
const response = await fetch("https://example.com/files/sample.arw")
const blob = await response.blob()
const rawFile = new File([blob], "sample.arw", { type: blob.type })

// Import the RAW file bytes into the pipeline
await renderer.import(rawFile)

// Apply denoise 
renderer.setAdjustments({
	luminanceNoiseReduction: 0.4
	colorNoiseReduction: 0.5
})

// When ready, export data as JPEG bytes
const outputJPEGBlob = await renderer.export({
    format: "jpeg"
})

// The blob now contains denoised JPEG file converted from RAW file 
```

## Local Healing Mask 

`npm install @polarr-next/mask-healing`

Adds an ability to specify local image data regions to be filled with other regions within the same image (heal certain image spots).

The following new interfaces are available in this module:

```typescript
// Healing rectangle is in normalised coordinates
export interface HealingRectangle {
	x: number
	y: number
	width: number
	height: number
}

export interface HealingAdjustment {
	source: HealingRectangle 
	destination: HealingRectangle
}
```

These items can be specified as part of adjustments when `setAdjustments` is called:

```typescript
export interface HealingAdjustments {
    healingAdjustmnets: HealingAdjustment[]
}
```

### Apply Mixed Adjustments

Here is an example of applying healing mixed with other global and local adjustments to the image:

```typescript
import { Renderer } from "@polarr-next"
import { JPEGIOAdapter } from "@polarr-next/jpeg-raw"
import { AdjustmentsPipeline } from "@polarr-next/adjustments"
import { DenoisePipeline } from "@polarr-next/denoise"
import { HealingPipeline } from "@polarr-next/healing" 

// Create an offscreen renderer with a managed context
const renderer = new Renderer()

// Register all modules
renderer.use(JPEGIOAdapter)
renderer.use(AdjustmentsPipeline) 
renderer.use(DenoisePipeline)
renderer.use(HealingPipeline)

// Obtain (or create) File with one of supported RAW formats (for example, .jpg file)
const response = await fetch("https://example.com/files/some-photo.jpg")
const blob = await response.blob()
const jpgFile = new File([blob], "some-photo.jpg", { type: blob.type })

// Import the RAW file bytes into the pipeline
await renderer.import(jpgFile)

renderer.setAdjustments({
    // Some global adjustments
    exposure: -0.1,
    highlights: -0.12,
    shadows: 0.15,
    clarity: 0.2,
    dehaze: 0.1

    // One curve for all colors
    curvesAll: [
	[0, 0], [128, 105], [255, 255]
    ]

    // Replace reds with lighter more saturated greens using point color
    pointColors: [
       	{
		color: [0.7, 0.1, 0.1],
		hueShift: 0.5,
		satShift: 0.1,
		lumShift: 0.2,
		range: 0.5
	}
    ],

    // Color Grading
    gradingHighlightsHue: 0.3,
    gradingHighlightsSaturation: -0.3,
    gradingHighlightsLuminance: 0.2,

    // Denoise
    luminanceNoiseReduction: 0.4
    colorNoiseReduction: 0.5

    // One healing patch to remove, for example, a socket in the wall
    healingAdjustments: [
        {
            source: {
                x: 0.4, 
                y: 0.2,
                width: 0.1,
                height: 0.1
            },
            destination: {
                x: 0.6, 
                y: 0.35,
                width: 0.1,
                height: 0.1
            }
        }
    ],

    {
            // Radial mask with some additional adjustments
            active: true,
            masks: [
                // Slightly feathered radial mask
                {
                    blendMode: "add",
                    active: true,
                    inverted: false,
                    type: "radial",
		    // Mask is at the center of the image
                    centerX: 0.5,
                    centerY: 0.5,
                    radiusX: 0.1,
                    radiusY: 0.2,
                    angle: 0,
                    feather: 0.2
                },
		{
		    // Diagonal brush strokes from the corner of the image towards the center
		    blendMode: "add",
		    type: "brush",
		    strokes: [{
			// Points are specified by pairs (X, Y, X, Y, etc)
			points: [0.0, 0.0, 0.1, 0.1, 0.2, 0.2, 0.3, 0.3],
			feather: 0.3,
			size: 0.4,
			spacing: 0.1,
		    }],
		    active: true,
		    inverted: false,
		    blendMode: "add",
		},
            ],
            // Group adjustments that are only applied to the radial mask
            exposure: 0.2,
            contrast: 0.11
    }
})

// When ready, export data as PNG bytes
const outputJPEGBlob = await renderer.export({
    format: "jpeg",
    quality: 0.9
})

// Download, save, or send the blob whenever
```

The specified destination region will be filled with pixel data from the source region. 

## AI Local Subject Mask

`npm install @polarr-next/ai-mask-subject`

Adds an ability to add new `SubjectMask` and new `BackgroundMask` (inverted `SubjectMask`) as part of local adjustments with the following definition:

```typescript
export interface SubjectMask extends BaseMask {
	type: "subject"
}

export interface BackgroundMask extends BaseMask {
	type: "background"
}
```

### Adjust Subject And Background

Here is an example of getting the mask directly from the `@polarr-next/ai-mask-subject` library:

```typescript
// This line of code imports the module to process AI Mask Subject
// The module contains the Tensorflow model that is loaded to make processing possible.
import { Renderer } from "@polarr-next"
import { AdjustmentsSubjectMask } from "@polarr-next/ai-mask-subject"

// Create an offscreen renderer with a managed context
const renderer = new Renderer({
	// It is possible to pass in the existing GL context to work with 
	gl
})

// Create the subject mask model instance
const subjectMaskModel = new AdjustmentsSubjectMask(renderer)

// Set up the model (wait for the underlying Tensorflow model to be loaded)
await subjectMaskModel.load()

// Get the RAW mask data (0, 1) for the input texture
// The texture can be either RGBA8 texture or RGBA16F texture with an image data.
const mask = await subjectMaskModel.getMask({
	texture
})

// The `mask` is an ArrayBuffer instance for 512x512 computed subject mask.
```

Here is an example of adjusting subject and background of the image at the same time using AI matting provided by `@polarr-next/ai-mask-subject`:

```typescript
import { Renderer } from "@polarr-next"
import { RawIOAdapter } from "@polarr-next/io-raw"
import { JPEGIOAdapter } from "@polarr-next/png-raw"
import { AdjustmentsPipeline } from "@polarr-next/adjustments"

// This line of code imports the module to process AI Mask Subject
// The module contains the Tensorflow model that is loaded to make processing possible.
import { AdjustmentsSubjectMask } from "@polarr-next/ai-mask-subject"

// Create an offscreen renderer with a managed context
const renderer = new Renderer()

// Register all modules
renderer.use(RawIOAdapter)
renderer.use(JPEGIOAdapter)
renderer.use(AdjustmentsPipeline)

// This line makes sure that the Tensorflow model from the @polarr-next/ai-mask-subject module is available to renderer
renderer.use(AdjustmentsSubjectMask)

// Obtain (or create) File with one of supported RAW formats (for example, .arw file)
const response = await fetch("https://example.com/files/photo-with-a-subject.arw")
const blob = await response.blob()
const rawFile = new File([blob], "photo-with-a-subject.arw", { type: blob.type })

// Import the RAW file bytes into the pipeline
await renderer.import(rawFile)

// Adjust both subject and background
renderer.setAdjustments({
    // Global adjustment
    exposure: -0.1,

    // Local adjustments
    localAdjustments: [
        // Set specific adjustments to image foreground (subject only)
        {
            active: true,
            masks: [
                {
                    blendMode: "add",
                    active: true,
                    inverted: false,
                    type: "subject",
                }
            ],
            // Group adjustments
            highlights: 0.2,
            contrast: 0.11
        }

        // Set specific adjustments to image background (everything excluding subject):
        {
            active: true,
            masks: [
                {
                    blendMode: "add",
                    active: true,
                    inverted: false,
                    type: "background",
                }
            ],
            // Group adjustments
            highlights: -0.4,
            contrast: 0.11,
            saturation: -0.7
        }

    ]
})

// When ready, export data as PNG bytes
const outputJPEGBlob = await renderer.export({
    format: "jpeg",
    quality: 0.9
})

// Download, save, or send the blob whenever
```

Internally, Tensorflow models are used to extract the subject from the image whenever the local mask with `background` or `foreground` types are specified.

## AI Local Sky Mask

`npm install @polarr-next/ai-mask-sky`

Adds an ability to add new `SkyMask` as part of local adjustments with the following definition:

```typescript
export interface SkyMask extends BaseMask {
	type: "sky"
}
```

Usage:

```typescript
renderer.setAdjustments({
    // Global adjustment
    saturation: -0.2,

    // Local subject adjustment
    localAdjustments: [
        {
            active: true,
            masks: [
                {
                    blendMode: "add",
                    active: true,
                    inverted: false,
                    type: "sky",
                }
            ],
            // Group adjustments
            saturation: 0.3
        }
    ]
})
```

## AI Auto Straighten 

`npm install @polarr-next/ai-straighten`

Adds an ability to automatically calculate straighten value for the currently loaded image data.

Usage:

```typescript
await renderer.computeAdjustments(["straighten"])

console.log("Auto straighten is now:", renderer.adjustments.cropStraighten)
```

After calling the method above, the following properties will be auto filled in current image data `Adjustments`:

```typescript
{
	cropStraighten: number
}
```

## AI Auto Lighting & White Balance 

`npm install @polarr-next/ai-lighting`

Adds an ability to automatically calculate lighting and white-balance values for the currently loaded image data.

Usage:

```typescript
await renderer.computeAdjustments(["lighting"])

console.log(
    "Tint and temperature are now:", 
    renderer.adjustments.tint, 
    renderer.adjustments.temperature
)
```

After calling the method above, the following properties will be auto filled in current image data `Adjustments`:

```typescript
{
	tint: number
    temperature: number
    exposure: number
    highlights: number
    shadows: number
}
```

## AI Auto Batch Adjustments

`npm install @polarr-next/ai-batch-adjustments`

> _This module depends on the Adjustments module `@polarr-next/adjustments` to be installed._

The module provides a way to seamlessly transfer adjustments from one photo which is marked a key photo to other photos.

The module implicitly groups the photos and then, if possible, transfers adjustments to these photos.

The initialization process is slightly different from the rest of the modules. This creates a group that will manage processing of multiple images sequentially:

```typescript
import { Renderer } from "@polarr-next"
import { RawIOAdapter } from "@polarr-next/io-raw"
import { PNGIOAdapter } from "@polarr-next/png-raw"
import { AdjustmentsPipeline } from "@polarr-next/adjustments"
import { AILightingPipeline } from "@polarr-next/ai-lighting"
import { AutoAdjustmentsGroup } from "@polarr-next/ai-batch-adjustments"

// Create an offscreen renderer with a managed context
const renderer = new Renderer()

// Register all modules for the base renderer
renderer.use(RawIOAdapter)
renderer.use(PNGIOAdapter)
renderer.use(AdjustmentsPipeline) 
renderer.use(AILightingPipeline)

// Somehow get the list of images to be processed.
const files = await getAllRAWFilesToProcess()

// Create the auto adjustments group to be able to process photos in a batch
const group = new AutoAdjustmentsGroup({
    // The copy of the renderer will be created and managed by the adjustments group to do internal processing and rendering
    renderer,
    // Create a mapping of entries with ids
    entries: files.map((file) => { id: file.name, file }),
    // Specify which auto adjustments must be computed
    computedAdjustments: ["lighting"],
    // Subscribe to changes to the whole queue
    onQueueStateChange: (state) => {
        // Get information about the state
        console.log("Processed photos in a group: ", state.count, "out of", state.total)
    },
    // Subscribe to changes to single photo by its id
    onPhotoStateChange: (state) => {
        // Get information about the photo
        console.log("Photo", state.id, "status changed to", state.status)

        // Update application UI state on status changes

        if (state.status === "completed") {
            // Photo has been processed
            // It is now possible to get adjustments for this photo via
            console.log("Photo", state.id, "adjustments are now:", group.getAdjustments(state.id))
        }
        else if (state.status === "pending") {
            // Photo is not yet processed
        }
        else if (state.status === "processing") {
            // Photo is being processed right now
        }
        
    }
})

// Resume processing of newly added photos
group.resume()
```

The `computedAdjustments` option is related to the instance method `Renderer.computeAdjustments` which will be called on each photo being processing inside the group. For the example above it means that each photo will have auto lighting and white balance computed for them before starting the processing.

Initially, when the group just started processing, all photos will be split into clusters using AI. Then adjustments from one photos, if exist, will be transferred into adjustments to other photos.

When necessary, pause processing which can be later resumed:

```typescript
// Halt processing of all photos
group.pause()
```

It is also possible to mark certain photos as references. This triggers reprocessing of a queue internally for the photos that are close to the one marked as a reference

```typescript
// Set certain photos adjustments to be transferred to similar photos
group.setAdjustments("some-photo-id", {
    exposure: -0.1,
    highlights: 0.15
})

// Mark certain photo as a reference
group.markAsReference("some-photo-id")
```

The queue will be restarted with the photo just marked as a reference. The similar photos will receive similar adjustments (after computed adjustments applied if necessary).

It is possible to retrieve the adjustments for photos that are already processed:

```typescript
// Get adjustments of photo that was already processed
const myAdjustments = group.getAdjustments("some-photo-id")

if (myAdjustments) {
    // Adjustments exist, it is now possible to use them for the current renderer
}
else {
    // It is too early, the adjustments don't exist yet
}
```

> It is advisable to always keep the `AutoAdjustmentsGroup` instance in a worker so its processing does not block the UI.

## Personalised AI Style 

When working with the `@polarr-next/ai-batch-adjustments` module, it is also possible to store all applied adjustments (by user and automatically) as a style that can be saved and applied later on. The style contains all information that is necessary to reapply color adjustments to any group of photos.

Here is an example that features saving a style and applying it between groups of photos:

```typescript
import { Renderer } from "@polarr-next"
import { RawIOAdapter } from "@polarr-next/io-raw"
import { PNGIOAdapter } from "@polarr-next/png-raw"
import { AdjustmentsPipeline } from "@polarr-next/adjustments"
import { AILightingPipeline } from "@polarr-next/ai-lighting"
import { AutoAdjustmentsGroup } from "@polarr-next/ai-batch-adjustments"

// Create an offscreen renderer with a managed context
const renderer = new Renderer()

// Register all modules for the base renderer
renderer.use(RawIOAdapter)
renderer.use(PNGIOAdapter)
renderer.use(AdjustmentsPipeline) 
renderer.use(AILightingPipeline)

// Get the first group of RAW files to process (via network or local file system)
const firstGroupFiles = await getAllRAWFilesToProcess()

// The AutoAdjustmentsGroup is essentially storage for all adjustments
// Once the AutoAdjustmentsGroup.resume instance method is called, the adjustments are computed
// And stored inside the AutoAdjustmentsGroup. 
const firstGroup = new AutoAdjustmentsGroup({
    renderer,
    entries: firstGroupFiles.map((file) => { id: file.name, file }),
    computedAdjustments: ["lighting"]
})

// Adjust some example file manually
firstGroup.setAdjustments("special-file-id", {
	exposure: -0.1,
	saturation: 0.2,
	highlights: 0.1
})

// Mark the example file as a reference
firstGroup.markAsReference("special-file-id")

// Start processing first group photos
firstGroup.resume()

// Wait for all photos to be processed in a group
await firstGroup.waitUntilCompleted()

// Compute the AI style to be saved for later
// This returns a Blob that can be saved as a file on a filesystem or in a database
// The AI style is a set of rules that can be applied to any AutoAdjustmentsGroup instance.
// It doesn't specify exact adjustments for photos, but rather creates smart representation of mixed AI adjustments
// and auto computed adjustments.
const aiStyle = await firstGroup.saveAIStyle()

// Later on, get the second batch of photos, different from the first batch and also process it
// Get the first group of RAW files to process (via network or local file system)
const secondGroupFiles = await getMoreRAWFilesToProcess()

const secondGroup = new AutoAdjustmentsGroup({
    renderer,
    entries: secondGroupFiles.map((file) => { id: file.name, file }),
    computedAdjustments: ["lighting"]
})

// Load previously saved AI style with a set of predefined smart adjustment rules
// The smart adjustments applied will be unique to each image based on image characteristics (ISO, exposure, portrait/landscape) 
secondGroup.loadAIStyle(aiStyle)

// Start processing and wait until it finished
secondGroup.resume()
await secondGroup.waitUntilCompleted()

// Process all images with adjustments applied to them sequentially
for (const file in secondGroupFiles) {
	await renderer.import(file)
	// Adjustments obtained from secondGroup are *unique* to each image
	// The whole aiStyle preservers the overall look&feel of all adjustments from firstGroupFiles and tries to
	// add the same look&feel to the secondGroupFiles.
	renderer.setAdjustments(secondGroup.adjustments(file.name))
	const jpegBlob = await renderer.export({ format: "jpeg" )

	// Save JPEG blob somewhere on the filesystem or send via network
}
```
