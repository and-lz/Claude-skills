# iOS 26 Media, Camera & On-Device Intelligence

Comprehensive reference for camera capture, photo/video processing, on-device ML, augmented reality, and PaperKit.

## Table of contents

1. [AVFoundation camera capture](#1-avfoundation-camera-capture)
2. [PhotoKit & PHPicker](#2-photokit--phpicker)
3. [Photo & video editing](#3-photo--video-editing)
4. [CoreML integration](#4-coreml-integration)
5. [Vision framework](#5-vision-framework)
6. [Natural Language framework](#6-natural-language-framework)
7. [ARKit & RealityKit](#7-arkit--realitykit)
8. [PaperKit (NEW in iOS 26)](#8-paperkit-new-in-ios-26)
9. [Anti-patterns](#9-anti-patterns)

---

## 1. AVFoundation camera capture

### Setup pattern in SwiftUI

Wrap `AVCaptureSession` in a `UIViewController`, then bridge to SwiftUI:

```swift
import AVFoundation
import SwiftUI

class CameraViewController: UIViewController {
    let captureSession = AVCaptureSession()
    private let photoOutput = AVCapturePhotoOutput()

    override func viewDidLoad() {
        super.viewDidLoad()

        guard let device = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back),
              let input = try? AVCaptureDeviceInput(device: device) else { return }

        captureSession.beginConfiguration()
        captureSession.sessionPreset = .photo
        if captureSession.canAddInput(input) { captureSession.addInput(input) }
        if captureSession.canAddOutput(photoOutput) { captureSession.addOutput(photoOutput) }
        captureSession.commitConfiguration()

        let previewLayer = AVCaptureVideoPreviewLayer(session: captureSession)
        previewLayer.videoGravity = .resizeAspectFill
        previewLayer.frame = view.bounds
        view.layer.addSublayer(previewLayer)

        Task.detached { [captureSession] in captureSession.startRunning() }
    }

    func capturePhoto(delegate: AVCapturePhotoCaptureDelegate) {
        let settings = AVCapturePhotoSettings()
        photoOutput.capturePhoto(with: settings, delegate: delegate)
    }
}

struct CameraView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> CameraViewController {
        CameraViewController()
    }
    func updateUIViewController(_ uiViewController: CameraViewController, context: Context) {}
}
```

### Permission flow

- Info.plist: `NSCameraUsageDescription` (required)
- Check: `AVCaptureDevice.authorizationStatus(for: .video)`
- Request: `await AVCaptureDevice.requestAccess(for: .video)`
- States: `.notDetermined`, `.restricted`, `.denied`, `.authorized`

### Camera controls

| Feature | API |
|---------|-----|
| Front/back switch | `AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .front/.back)` |
| Flash | `AVCapturePhotoSettings().flashMode = .auto/.on/.off` |
| Zoom | `device.videoZoomFactor = 2.0` (within `device.minAvailableVideoZoomFactor...device.maxAvailableVideoZoomFactor`) |
| Focus | `device.focusPointOfInterest = point; device.focusMode = .autoFocus` |
| Exposure | `device.exposurePointOfInterest = point; device.exposureMode = .autoExpose` |

### Video recording

```swift
let movieOutput = AVCaptureMovieFileOutput()
captureSession.addOutput(movieOutput)

// Start recording
let url = FileManager.default.temporaryDirectory.appendingPathComponent("video.mov")
movieOutput.startRecording(to: url, recordingDelegate: self)

// Stop recording
movieOutput.stopRecording()
```

### iOS 26 additions

**AVCaptureControl hierarchy** — new abstract base class with system-defined subclasses:
- System zoom and exposure bias controls
- Generic continuous/discrete sliders and pickers
- Add controls to `AVCaptureSession`, respond via action handlers or KVO

**AVCaptureEventInteraction** — control physical button presses (volume buttons) in camera apps:
- Event phases: `.began`, `.cancelled`, `.ended`
- Map physical gestures to capture actions

**AirPods remote camera** — trigger capture from AirPods controls.

---

## 2. PhotoKit & PHPicker

### PhotosPicker (SwiftUI — preferred)

```swift
import PhotosUI
import SwiftUI

struct PhotoPickerView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var image: Image?

    var body: some View {
        PhotosPicker(selection: $selectedItem, matching: .images) {
            Label("Select Photo", systemImage: "photo")
        }
        .onChange(of: selectedItem) { _, newItem in
            Task {
                if let data = try? await newItem?.loadTransferable(type: Data.self),
                   let uiImage = UIImage(data: data) {
                    image = Image(uiImage: uiImage)
                }
            }
        }
    }
}
```

### PHPickerViewController (UIKit — for advanced config)

```swift
var config = PHPickerConfiguration()
config.filter = .any(of: [.images, .videos, .livePhotos])
config.selectionLimit = 5
config.selection = .ordered
let picker = PHPickerViewController(configuration: config)
```

- No photo library permission required — privacy-preserving by design
- Returns `PHPickerResult` with `NSItemProvider`

### PHAsset fetching (requires permission)

```swift
import Photos

let options = PHFetchOptions()
options.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]
options.fetchLimit = 50
let assets = PHAsset.fetchAssets(with: .image, options: options)

// Request image
PHImageManager.default().requestImage(
    for: asset,
    targetSize: CGSize(width: 300, height: 300),
    contentMode: .aspectFill,
    options: nil
) { image, info in
    // Use image
}
```

### Change observation

```swift
class PhotoLibraryObserver: NSObject, PHPhotoLibraryChangeObserver {
    func photoLibraryDidChange(_ changeInstance: PHChange) {
        // React to library changes
    }
}
PHPhotoLibrary.shared().register(observer)
```

### Limited access (iOS 14+)

- `PHAccessLevel.addOnly` — write-only, no browsing
- `PHAccessLevel.readWrite` — full access or limited selection
- `PHPhotoLibrary.shared().presentLimitedLibraryPicker(from:)` — let user expand selection

### iOS 26

- `PHBackgroundResourceUploadExtension` — background photo upload extension for cloud sync apps

---

## 3. Photo & video editing

### CIFilter chains

```swift
import CoreImage
import CoreImage.CIFilterBuiltins

let context = CIContext()
let ciImage = CIImage(image: originalImage)!

let colorControls = CIFilter.colorControls()
colorControls.inputImage = ciImage
colorControls.saturation = 1.2
colorControls.brightness = 0.05
colorControls.contrast = 1.1

let vignette = CIFilter.vignette()
vignette.inputImage = colorControls.outputImage
vignette.intensity = 0.5
vignette.radius = 2.0

if let output = vignette.outputImage,
   let cgImage = context.createCGImage(output, from: output.extent) {
    let result = UIImage(cgImage: cgImage)
}
```

### Non-destructive photo editing

- `PHContentEditingInput` — original asset data
- `PHContentEditingOutput` — edited result with adjustment data
- `PHPhotoLibrary.shared().performChanges { ... }` to save

### Video editing

- `AVVideoComposition` — apply filters per-frame
- `AVMutableComposition` — combine clips, add audio tracks
- `AVAssetExportSession` — export to file with preset quality

---

## 4. CoreML integration

### Loading and predicting

```swift
import CoreML

let config = MLModelConfiguration()
config.computeUnits = .all // .cpuOnly, .cpuAndGPU, .cpuAndNeuralEngine

let model = try MyImageClassifier(configuration: config)
let prediction = try model.prediction(image: pixelBuffer)
print(prediction.classLabel, prediction.classLabelProbs)
```

### Compute units

| Option | Best for |
|--------|----------|
| `.cpuOnly` | Small models, compatibility |
| `.cpuAndGPU` | Medium models, real-time inference |
| `.cpuAndNeuralEngine` | Large models, best perf on A11+ |
| `.all` | Let system decide (recommended default) |

### Combined with Vision

```swift
let vnModel = try VNCoreMLModel(for: model.model)
let request = VNCoreMLRequest(model: vnModel) { request, error in
    guard let results = request.results as? [VNClassificationObservation] else { return }
    let top = results.prefix(3)
}
let handler = VNImageRequestHandler(cgImage: image)
try handler.perform([request])
```

### CoreML vs Foundation Models

| Use CoreML when | Use Foundation Models when |
|-----------------|---------------------------|
| Specialized trained model (image classification, object detection, style transfer) | General-purpose text generation/understanding |
| Custom domain model (.mlmodel) | Summarization, classification, Q&A |
| Real-time inference on camera frames | Structured data extraction |
| Deterministic, reproducible output | Conversational, generative output |

### Best practices

- Compile models at build time (Xcode auto-compiles .mlmodel)
- Use quantized models to reduce size (Float16, Int8)
- Use On-Demand Resources for large models
- Profile with Instruments → CoreML template

---

## 5. Vision framework

### Text recognition (OCR)

```swift
import Vision

let request = VNRecognizeTextRequest { request, error in
    guard let observations = request.results as? [VNRecognizedTextObservation] else { return }
    let strings = observations.compactMap { $0.topCandidates(1).first?.string }
    let fullText = strings.joined(separator: "\n")
}
request.recognitionLevel = .accurate // .fast for real-time
request.recognitionLanguages = ["en-US"]
request.usesLanguageCorrection = true

let handler = VNImageRequestHandler(cgImage: image, options: [:])
try handler.perform([request])
```

### Face detection

```swift
let faceRequest = VNDetectFaceRectanglesRequest { request, error in
    guard let faces = request.results as? [VNFaceObservation] else { return }
    for face in faces {
        let boundingBox = face.boundingBox // normalized coordinates
    }
}

// For landmarks (eyes, nose, mouth):
let landmarkRequest = VNDetectFaceLandmarksRequest()
```

### Saliency

```swift
// Attention-based: where humans naturally look
let attentionRequest = VNGenerateAttentionBasedSaliencyImageRequest()

// Objectness-based: where distinct objects are
let objectnessRequest = VNGenerateObjectnessBasedSaliencyImageRequest()
```

### Live camera analysis

```swift
// Implement AVCaptureVideoDataOutputSampleBufferDelegate
func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
    guard let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else { return }

    let handler = VNImageRequestHandler(cvPixelBuffer: pixelBuffer, options: [:])
    try? handler.perform([textRequest, faceRequest])
}
```

### Key request types

| Request | Purpose |
|---------|---------|
| `VNRecognizeTextRequest` | OCR |
| `VNDetectFaceRectanglesRequest` | Face bounding boxes |
| `VNDetectFaceLandmarksRequest` | Facial feature points |
| `VNDetectBarcodesRequest` | QR codes, barcodes |
| `VNClassifyImageRequest` | Scene classification |
| `VNGenerateAttentionBasedSaliencyImageRequest` | Visual attention heatmap |
| `VNTrackObjectRequest` | Real-time object tracking |
| `VNDetectHumanBodyPoseRequest` | Body pose estimation |

---

## 6. Natural Language framework

### Tokenization

```swift
import NaturalLanguage

let tokenizer = NLTokenizer(unit: .word)
tokenizer.string = "The quick brown fox jumps."
tokenizer.enumerateTokens(in: text.startIndex..<text.endIndex) { range, _ in
    print(text[range])
    return true
}
```

### Part-of-speech tagging and NER

```swift
let tagger = NLTagger(tagSchemes: [.lexicalClass, .nameType])
tagger.string = "Apple CEO Tim Cook visited Paris in March."

// Parts of speech
tagger.enumerateTags(in: text.startIndex..<text.endIndex, unit: .word, scheme: .lexicalClass) { tag, range in
    if let tag { print("\(text[range]): \(tag.rawValue)") }
    return true
}

// Named entities
tagger.enumerateTags(in: text.startIndex..<text.endIndex, unit: .word, scheme: .nameType) { tag, range in
    if let tag { print("\(text[range]): \(tag.rawValue)") } // PersonalName, PlaceName, OrganizationName
    return true
}
```

### Sentiment analysis

```swift
let tagger = NLTagger(tagSchemes: [.sentimentScore])
tagger.string = "I love this amazing product!"
let sentiment = tagger.tag(at: text.startIndex, unit: .paragraph, scheme: .sentimentScore)
// sentiment?.rawValue is a Double string from -1.0 (negative) to 1.0 (positive)
```

### Language identification

```swift
let recognizer = NLLanguageRecognizer()
recognizer.processString("Bonjour, comment allez-vous?")
let language = recognizer.dominantLanguage // .french
let hypotheses = recognizer.languageHypotheses(withMaximum: 3) // ranked
```

### Embeddings (semantic similarity)

```swift
if let embedding = NLEmbedding.wordEmbedding(for: .english) {
    let distance = embedding.distance(between: "dog", and: "puppy") // small distance = similar
    let neighbors = embedding.neighbors(for: "king", maximumCount: 5) // similar words
}
```

### NL framework vs Foundation Models

| Use NL framework when | Use Foundation Models when |
|-----------------------|---------------------------|
| Fast, deterministic classification | Generative text tasks |
| Tokenization, POS tagging, NER | Complex reasoning |
| Sentiment scoring | Conversational interaction |
| Language detection | Summarization, rewriting |
| Word similarity/embeddings | Structured data extraction |

---

## 7. ARKit & RealityKit

### RealityView (SwiftUI — iOS 18+)

```swift
import RealityKit
import SwiftUI

struct ARContentView: View {
    var body: some View {
        RealityView { content in
            // Load 3D model
            if let model = try? await ModelEntity(named: "robot.usdz") {
                model.position = [0, 0, -1] // 1 meter in front
                model.scale = [0.1, 0.1, 0.1]
                content.add(model)
            }
        }
    }
}
```

### ARView (UIKit-based)

```swift
import ARKit
import RealityKit

let arView = ARView(frame: .zero)
let config = ARWorldTrackingConfiguration()
config.planeDetection = [.horizontal, .vertical]
config.environmentTexturing = .automatic
arView.session.run(config)

// Add anchor and model
let anchor = AnchorEntity(.plane(.horizontal, classification: .floor, minimumBounds: [0.2, 0.2]))
let model = try! ModelEntity.loadModel(named: "chair")
anchor.addChild(model)
arView.scene.addAnchor(anchor)
```

### Anchor types

| Anchor | Use case |
|--------|----------|
| `AnchorEntity(.plane(.horizontal))` | Place objects on floors/tables |
| `AnchorEntity(.plane(.vertical))` | Place objects on walls |
| `AnchorEntity(.face)` | Face filters/masks |
| `AnchorEntity(.image(group:name:))` | Image recognition triggers |
| `AnchorEntity(.body)` | Body tracking |

### Materials

- `SimpleMaterial(color: .blue, isMetallic: true)` — basic
- `PhysicallyBasedMaterial()` — PBR with roughness, metallic, etc.
- `UnlitMaterial(color: .white)` — no lighting
- `ShaderGraphMaterial` — custom Reality Composer Pro shaders

### Object capture (iOS 17+)

Photogrammetry pipeline for creating 3D models from photos.

### ARCoachingOverlayView

System UI that guides users to move device for plane detection.

### iOS 26 (visionOS 26)

- Direct ARKit data access for anchoring
- Collision and physics for real-world objects
- 90Hz hand tracking for immersive visionOS apps (no code changes needed)

---

## 8. PaperKit (NEW in iOS 26)

### Overview

PaperKit powers Apple's markup experience system-wide (Notes, Screenshots, QuickLook, Journal). Built on PencilKit + PDFKit.

### Features

- Canvas for drawing and markup elements
- Shapes, images, textboxes, and more
- Stickers, links, and annotations

### Platform APIs

| Platform | API |
|----------|-----|
| iOS / iPadOS / visionOS | `MarkupEditViewController` — insertion menu for markup elements |
| macOS | `MarkupToolbarViewController` — toolbar with drawing tools and annotation buttons |

### Integration

PaperKit extends PencilKit canvases with structured markup capabilities. Integrate when building note-taking, document annotation, or drawing features.

---

## 9. SpeechAnalyzer (NEW in iOS 26 — replaces SFSpeechRecognizer)

SpeechAnalyzer is the new modular speech-to-text API. It supports long-form audio, on-device processing, automatic language detection, and low-latency live transcription.

### Core classes

| Class | Purpose |
|-------|---------|
| `SpeechAnalyzer` | Coordinator — manages modules and audio pipeline |
| `SpeechTranscriber` | Speech-to-text module (attach to analyzer) |
| `SpeechDetector` | Voice activity detection module |

### Live transcription setup

```swift
import Speech

// 1. Create transcriber and analyzer
let transcriber = SpeechTranscriber(
    locale: .current,
    preset: .progressiveLiveTranscription  // .offlineTranscription for files
)
let analyzer = SpeechAnalyzer(modules: [transcriber])

// 2. Get optimal audio format
let format = await SpeechAnalyzer.bestAvailableAudioFormat(compatibleWith: [transcriber])

// 3. Ensure language model is downloaded
if let downloader = try await AssetInventory.assetInstallationRequest(supporting: [transcriber]) {
    try await downloader.downloadAndInstall()
}

// 4. Start processing audio
let (inputSequence, inputBuilder) = AsyncStream<AnalyzerInput>.makeStream()
try await analyzer.start(inputSequence: inputSequence)

// 5. Feed audio buffers (from AVAudioEngine)
audioEngine.inputNode.installTap(onBus: 0, bufferSize: 4096, format: micFormat) { buffer, _ in
    let converted = try! converter.convertBuffer(buffer, to: format)
    inputBuilder.yield(AnalyzerInput(buffer: converted))
}
try audioEngine.start()

// 6. Consume results
for try await result in transcriber.results {
    if result.isFinal {
        finalText += result.text      // Committed, won't change
    } else {
        volatileText = result.text    // May be revised — show in lighter color
    }
}
```

### File transcription

```swift
let transcriber = SpeechTranscriber(locale: .current, preset: .offlineTranscription)
let analyzer = SpeechAnalyzer(modules: [transcriber])

async let transcription = transcriber.results
    .reduce("") { text, result in text + result.text }

if let lastSample = try await analyzer.analyzeSequence(from: audioFileURL) {
    try await analyzer.finalizeAndFinish(through: lastSample)
}
let fullText = try await transcription
```

### Volatile vs final results

| Type | Accuracy | When to use |
|------|----------|-------------|
| Volatile (`!result.isFinal`) | Lower, continuously revised | Live preview — show in lighter/italic style |
| Final (`result.isFinal`) | Highest with full context | Persist, save, send — won't change |

### Reporting and attribute options

```swift
let transcriber = SpeechTranscriber(
    locale: .current,
    reportingOptions: [.volatileResults],     // also get in-progress results
    attributeOptions: [.audioTimeRange]        // include timestamps per segment
)
```

### Fallback for unsupported devices

```swift
if !SpeechTranscriber.supportsDevice() {
    // Fall back to DictationTranscriber (older, less capable)
    let dictation = DictationTranscriber(locale: .current)
}
```

### Key advantages over SFSpeechRecognizer
- **Long-form audio**: optimized for lectures, meetings, conversations
- **On-device**: complete privacy, no internet needed
- **Auto language management**: no user configuration required
- **Low latency**: real-time transcription
- **Distant audio**: works when speakers aren't close to mic

---

## 10. Image Playground & ImageCreator (Apple Intelligence)

Image Playground generates AI images on-device from text descriptions, existing images, or drawings. Two integration paths: UI sheet (user-facing) or programmatic API.

### 10.1 Image Playground sheet (user-facing UI)

```swift
import ImagePlayground

struct ContentView: View {
    @State private var showPlayground = false
    @State private var generatedImage: Image?

    var body: some View {
        Button("Create Image") { showPlayground = true }
            .imagePlaygroundSheet(
                isPresented: $showPlayground,
                concepts: [.text("A cat wearing a top hat")],
                style: .animation
            ) { result in
                generatedImage = Image(result.cgImage)
            }
    }
}
```

### 10.2 ImageCreator (programmatic, headless)

```swift
import ImagePlayground

func generateImage(prompt: String) async throws -> CGImage? {
    let creator = try await ImageCreator()

    // Check available styles
    guard creator.availableStyles.contains(.animation) else { return nil }

    let images = creator.images(
        for: [.text(prompt)],
        style: .animation,  // .animation, .illustration, .sketch
        limit: 1            // max 4 images
    )

    for try await image in images {
        return image.cgImage
    }
    return nil
}
```

### 10.3 Combining concepts

```swift
// Text + image input
let concepts: [ImagePlaygroundConcept] = [
    .text("A serene mountain landscape"),
    .image(referencePhoto)           // CGImage reference
]

let images = creator.images(for: concepts, style: .illustration, limit: 2)
```

### Available concept types

| Concept | Usage |
|---------|-------|
| `.text(String)` | Text description |
| `.image(CGImage)` | Reference image for style/content |
| `.drawing(PKDrawing)` | PencilKit drawing as input |
| `.extracted(from: CGImage)` | Extract subject from photo |

### Available styles

| Style | Description |
|-------|-------------|
| `.animation` | Pixar-like 3D animated style |
| `.illustration` | Flat illustration style |
| `.sketch` | Hand-drawn sketch style |

### Key constraints
- **Device requirement**: Apple Intelligence-compatible (A17 Pro+, M-series)
- **Availability check**: `try await ImageCreator()` throws `.notSupported` on ineligible devices
- **Max images**: 4 per request
- **On-device**: all generation runs locally, no data sent to servers
- **Styles**: iOS 26 adds ChatGPT-powered styles in addition to the 3 on-device styles

---

## 11. Anti-patterns

| Anti-pattern | Why | Instead |
|-------------|-----|---------|
| Use `UIImagePickerController` for photo selection | Deprecated API, limited features | Use `PhotosPicker` (SwiftUI) or `PHPickerViewController` |
| Request full photo library when only selecting | Unnecessary permission, user friction | Use PHPicker — requires no permission |
| Run Vision requests on main thread | Blocks UI, causes frame drops | Dispatch to background queue or use `.task { }` |
| Bundle huge unquantized ML models | Inflates app size, slow first load | Quantize (Float16/Int8), use On-Demand Resources |
| Ignore camera permission denial | Broken UX, no fallback | Show explanation + deep link to Settings |
| Missing Info.plist purpose strings | Crash on access, App Store rejection | Add `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription` |
| Synchronous PHImageManager in SwiftUI | Blocks rendering, choppy scrolling | Use async loading with placeholder |
| Create new AVCaptureSession per photo | Resource waste, slow startup | Reuse session, swap inputs/outputs as needed |
| Use Metal for simple image filters | Overkill, complex code | Use CIFilter — GPU-accelerated, much simpler |
