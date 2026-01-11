# PrintShot iOS App - Implementation Plan

**Version:** 1.0
**Target:** App Store Launch
**Timeline:** 12 weeks

---

## Overview

This plan breaks PrintShot into 6 phases with clear deliverables, dependencies, and technical guidance for each component.

```
Week  1-2   │ Phase 1: Foundation
Week  3-4   │ Phase 2: Camera & Crop
Week  5-7   │ Phase 3: Background Processing
Week  8-9   │ Phase 4: AI Tagging
Week 10-11  │ Phase 5: Export & Polish
Week 12     │ Phase 6: Launch Prep
```

---

## Phase 1: Foundation (Week 1-2)

### Goals
- Project architecture established
- Design system defined
- Core navigation working

### Tasks

#### 1.1 Project Setup (Day 1-2)

```
PrintShot/
├── App/
│   ├── PrintShotApp.swift          # Entry point
│   └── AppState.swift              # Global state
├── Features/
│   ├── Camera/
│   ├── Crop/
│   ├── Background/
│   ├── Tagging/
│   └── Export/
├── Core/
│   ├── Services/
│   ├── Models/
│   └── Extensions/
├── UI/
│   ├── Components/
│   └── Theme/
└── Resources/
    ├── Assets.xcassets
    └── Backgrounds/
```

**Decisions to lock:**
- Minimum iOS version: 16.0 (or 17.0 for better Vision APIs)
- Bundle ID: com.yourcompany.printshot
- Development team & signing

#### 1.2 Design System (Day 2-3)

```swift
// Theme.swift
enum Theme {
    enum Colors {
        static let background = Color("Background")
        static let surface = Color("Surface")
        static let primary = Color("Primary")
        static let textPrimary = Color("TextPrimary")
        static let textSecondary = Color("TextSecondary")
    }

    enum Spacing {
        static let xs: CGFloat = 4
        static let sm: CGFloat = 8
        static let md: CGFloat = 16
        static let lg: CGFloat = 24
        static let xl: CGFloat = 32
    }

    enum Radius {
        static let sm: CGFloat = 8
        static let md: CGFloat = 12
        static let lg: CGFloat = 20
    }
}
```

**Design assets needed from designer:**
- [ ] App icon (1024x1024 + all sizes)
- [ ] Color palette (light mode only for v1)
- [ ] Capture button design
- [ ] Background mode toggle/button design
- [ ] Tag chip component design

#### 1.3 Navigation Architecture (Day 3-4)

```swift
// AppState.swift
@Observable
class AppState {
    var currentScreen: Screen = .camera
    var capturedImage: UIImage?
    var croppedImage: UIImage?
    var processedImage: UIImage?
    var selectedTags: [String] = []

    enum Screen {
        case camera
        case crop
        case background
        case tagging
        case export
    }

    func reset() {
        currentScreen = .camera
        capturedImage = nil
        croppedImage = nil
        processedImage = nil
        selectedTags = []
    }
}
```

```swift
// ContentView.swift
struct ContentView: View {
    @State private var appState = AppState()

    var body: some View {
        switch appState.currentScreen {
        case .camera:
            CameraView(appState: appState)
        case .crop:
            CropView(appState: appState)
        case .background:
            BackgroundView(appState: appState)
        case .tagging:
            TaggingView(appState: appState)
        case .export:
            ExportView(appState: appState)
        }
    }
}
```

#### 1.4 Core Models (Day 4-5)

```swift
// Models/Capture.swift
import SwiftData

@Model
class Capture {
    var id: UUID
    var createdAt: Date
    var tags: [String]
    var backgroundMode: BackgroundMode
    var thumbnailData: Data?

    init(tags: [String] = [], backgroundMode: BackgroundMode = .preservePage) {
        self.id = UUID()
        self.createdAt = Date()
        self.tags = tags
        self.backgroundMode = backgroundMode
    }
}

enum BackgroundMode: String, Codable {
    case preservePage
    case replaceBackground
}

// Models/BackgroundOption.swift
struct BackgroundOption: Identifiable {
    let id: String
    let type: BackgroundType
    let color: Color?
    let imageName: String?

    enum BackgroundType {
        case solid
        case pattern
        case photo
    }
}
```

### Phase 1 Deliverable
- App launches
- Can navigate between placeholder screens
- Design tokens in place
- Models defined

---

## Phase 2: Camera & Crop (Week 3-4)

### Goals
- Full custom camera working
- Crop interface with draggable handles
- Image flows from camera → crop → next screen

### Tasks

#### 2.1 Camera Permissions (Day 1)

```swift
// Info.plist
NSCameraUsageDescription: "PrintShot needs camera access to capture pages from books and magazines."
```

```swift
// Services/CameraPermissions.swift
class CameraPermissions {
    static func request() async -> Bool {
        let status = AVCaptureDevice.authorizationStatus(for: .video)
        switch status {
        case .authorized:
            return true
        case .notDetermined:
            return await AVCaptureDevice.requestAccess(for: .video)
        default:
            return false
        }
    }
}
```

#### 2.2 Camera Service (Day 1-3)

```swift
// Services/CameraService.swift
import AVFoundation

@Observable
class CameraService: NSObject {
    private let session = AVCaptureSession()
    private let output = AVCapturePhotoOutput()
    private var device: AVCaptureDevice?

    var previewLayer: AVCaptureVideoPreviewLayer?
    var isFlashOn = false
    var capturedPhoto: UIImage?

    func configure() async throws {
        session.beginConfiguration()
        session.sessionPreset = .photo

        guard let camera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back) else {
            throw CameraError.noCameraAvailable
        }
        self.device = camera

        let input = try AVCaptureDeviceInput(device: camera)
        if session.canAddInput(input) {
            session.addInput(input)
        }

        if session.canAddOutput(output) {
            session.addOutput(output)
        }

        session.commitConfiguration()

        previewLayer = AVCaptureVideoPreviewLayer(session: session)
        previewLayer?.videoGravity = .resizeAspectFill
    }

    func start() {
        Task(priority: .userInitiated) {
            session.startRunning()
        }
    }

    func stop() {
        session.stopRunning()
    }

    func capture() {
        var settings = AVCapturePhotoSettings()
        settings.flashMode = isFlashOn ? .on : .off
        output.capturePhoto(with: settings, delegate: self)
    }

    func toggleFlash() {
        isFlashOn.toggle()
    }
}

extension CameraService: AVCapturePhotoCaptureDelegate {
    func photoOutput(_ output: AVCapturePhotoOutput,
                     didFinishProcessingPhoto photo: AVCapturePhoto,
                     error: Error?) {
        guard let data = photo.fileDataRepresentation(),
              let image = UIImage(data: data) else { return }

        Task { @MainActor in
            self.capturedPhoto = image
        }
    }
}
```

#### 2.3 Camera View (Day 3-4)

```swift
// Features/Camera/CameraView.swift
struct CameraView: View {
    @Bindable var appState: AppState
    @State private var cameraService = CameraService()
    @State private var showPermissionDenied = false

    var body: some View {
        ZStack {
            // Camera preview
            CameraPreviewView(cameraService: cameraService)
                .ignoresSafeArea()

            // Overlay UI
            VStack {
                // Top bar
                HStack {
                    Spacer()

                    Button(action: { cameraService.toggleFlash() }) {
                        Image(systemName: cameraService.isFlashOn ? "bolt.fill" : "bolt.slash")
                            .font(.title2)
                            .foregroundColor(.white)
                            .padding()
                    }

                    Button(action: { /* settings */ }) {
                        Image(systemName: "gear")
                            .font(.title2)
                            .foregroundColor(.white)
                            .padding()
                    }
                }

                Spacer()

                // Capture button
                CaptureButton(action: capture)
                    .padding(.bottom, 40)
            }
        }
        .task {
            await setupCamera()
        }
        .onChange(of: cameraService.capturedPhoto) { _, photo in
            if let photo {
                appState.capturedImage = photo
                appState.currentScreen = .crop
            }
        }
    }

    private func setupCamera() async {
        guard await CameraPermissions.request() else {
            showPermissionDenied = true
            return
        }
        try? await cameraService.configure()
        cameraService.start()
    }

    private func capture() {
        cameraService.capture()
    }
}

// Custom capture button
struct CaptureButton: View {
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            ZStack {
                Circle()
                    .fill(.white)
                    .frame(width: 72, height: 72)
                Circle()
                    .stroke(.white, lineWidth: 4)
                    .frame(width: 84, height: 84)
            }
        }
    }
}
```

#### 2.4 Crop Interface (Day 4-7)

```swift
// Features/Crop/CropView.swift
struct CropView: View {
    @Bindable var appState: AppState
    @State private var cropRect: CGRect = .zero
    @State private var imageSize: CGSize = .zero

    var body: some View {
        GeometryReader { geo in
            ZStack {
                Color.black.ignoresSafeArea()

                if let image = appState.capturedImage {
                    Image(uiImage: image)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .overlay(
                            CropOverlay(cropRect: $cropRect, bounds: geo.size)
                        )
                        .onAppear {
                            imageSize = geo.size
                            // Initialize crop rect to 80% of image
                            cropRect = CGRect(
                                x: geo.size.width * 0.1,
                                y: geo.size.height * 0.1,
                                width: geo.size.width * 0.8,
                                height: geo.size.height * 0.8
                            )
                        }
                }

                // Bottom controls
                VStack {
                    Spacer()

                    HStack(spacing: 40) {
                        Button("Retake") {
                            appState.currentScreen = .camera
                        }
                        .foregroundColor(.white)

                        Button(action: confirmCrop) {
                            Text("Confirm")
                                .fontWeight(.semibold)
                                .foregroundColor(.black)
                                .padding(.horizontal, 32)
                                .padding(.vertical, 12)
                                .background(Color.white)
                                .cornerRadius(Theme.Radius.md)
                        }
                    }
                    .padding(.bottom, 40)
                }
            }
        }
    }

    private func confirmCrop() {
        guard let image = appState.capturedImage else { return }
        appState.croppedImage = ImageProcessor.crop(image, to: cropRect, in: imageSize)
        appState.currentScreen = .background
    }
}

// Draggable crop overlay
struct CropOverlay: View {
    @Binding var cropRect: CGRect
    let bounds: CGSize

    @State private var activeHandle: Handle?

    enum Handle {
        case topLeft, topRight, bottomLeft, bottomRight
    }

    var body: some View {
        ZStack {
            // Dimmed area outside crop
            CropMask(rect: cropRect)
                .fill(.black.opacity(0.5))

            // Crop border
            Rectangle()
                .stroke(.white, lineWidth: 2)
                .frame(width: cropRect.width, height: cropRect.height)
                .position(x: cropRect.midX, y: cropRect.midY)

            // Corner handles
            ForEach([Handle.topLeft, .topRight, .bottomLeft, .bottomRight], id: \.self) { handle in
                CropHandle()
                    .position(handlePosition(for: handle))
                    .gesture(
                        DragGesture()
                            .onChanged { value in
                                updateCropRect(handle: handle, translation: value.translation)
                            }
                    )
            }
        }
    }

    private func handlePosition(for handle: Handle) -> CGPoint {
        switch handle {
        case .topLeft: return CGPoint(x: cropRect.minX, y: cropRect.minY)
        case .topRight: return CGPoint(x: cropRect.maxX, y: cropRect.minY)
        case .bottomLeft: return CGPoint(x: cropRect.minX, y: cropRect.maxY)
        case .bottomRight: return CGPoint(x: cropRect.maxX, y: cropRect.maxY)
        }
    }

    private func updateCropRect(handle: Handle, translation: CGSize) {
        // Handle drag logic with bounds checking
        // ... implementation details
    }
}

struct CropHandle: View {
    var body: some View {
        Circle()
            .fill(.white)
            .frame(width: 24, height: 24)
            .shadow(radius: 2)
    }
}
```

#### 2.5 Image Processor Service (Day 7-8)

```swift
// Services/ImageProcessor.swift
import CoreImage
import CoreImage.CIFilterBuiltins

class ImageProcessor {
    private static let context = CIContext()

    /// Crop image to specified rect
    static func crop(_ image: UIImage, to rect: CGRect, in viewSize: CGSize) -> UIImage? {
        // Convert view coordinates to image coordinates
        let scaleX = image.size.width / viewSize.width
        let scaleY = image.size.height / viewSize.height

        let imageRect = CGRect(
            x: rect.origin.x * scaleX,
            y: rect.origin.y * scaleY,
            width: rect.width * scaleX,
            height: rect.height * scaleY
        )

        guard let cgImage = image.cgImage?.cropping(to: imageRect) else { return nil }
        return UIImage(cgImage: cgImage, scale: image.scale, orientation: image.imageOrientation)
    }

    /// Extract dominant colors from image
    static func extractColors(from image: UIImage, count: Int = 5) -> [Color] {
        // Use Core Image or custom algorithm to extract palette
        // ... implementation
    }

    /// Add drop shadow to image
    static func addShadow(to image: UIImage, radius: CGFloat = 20, opacity: Float = 0.3) -> UIImage? {
        let shadowOffset = CGSize(width: 0, height: 10)
        let shadowColor = UIColor.black.withAlphaComponent(CGFloat(opacity))

        let size = CGSize(
            width: image.size.width + radius * 2,
            height: image.size.height + radius * 2 + shadowOffset.height
        )

        UIGraphicsBeginImageContextWithOptions(size, false, image.scale)
        defer { UIGraphicsEndImageContext() }

        guard let context = UIGraphicsGetCurrentContext() else { return nil }

        context.setShadow(offset: shadowOffset, blur: radius, color: shadowColor.cgColor)
        image.draw(at: CGPoint(x: radius, y: radius))

        return UIGraphicsGetImageFromCurrentImageContext()
    }
}
```

### Phase 2 Deliverable
- Camera captures photos
- Crop interface with working handles
- Image flows through to next screen
- Basic image processing utilities

---

## Phase 3: Background Processing (Week 5-7)

### Goals
- "Preserve Page" mode working
- "Replace Background" mode working
- Background picker (colors, patterns, photos)
- Shadow toggle

### Tasks

#### 3.1 Subject Extraction Service (Day 1-3)

```swift
// Services/SubjectExtractor.swift
import Vision

class SubjectExtractor {

    /// Extract subject from image using Vision framework (iOS 17+)
    @available(iOS 17.0, *)
    static func extractSubject(from image: UIImage) async throws -> UIImage? {
        guard let cgImage = image.cgImage else { return nil }

        let request = VNGenerateForegroundInstanceMaskRequest()
        let handler = VNImageRequestHandler(cgImage: cgImage)

        try handler.perform([request])

        guard let result = request.results?.first else { return nil }

        let maskPixelBuffer = try result.generateScaledMaskForImage(
            forInstances: result.allInstances,
            from: handler
        )

        return applyMask(maskPixelBuffer, to: image)
    }

    /// Fallback for iOS 16: Use person/object segmentation
    static func extractSubjectLegacy(from image: UIImage) async throws -> UIImage? {
        guard let cgImage = image.cgImage else { return nil }

        let request = VNGeneratePersonSegmentationRequest()
        request.qualityLevel = .accurate

        let handler = VNImageRequestHandler(cgImage: cgImage)
        try handler.perform([request])

        guard let result = request.results?.first,
              let maskBuffer = result.pixelBuffer else { return nil }

        return applyMask(maskBuffer, to: image)
    }

    private static func applyMask(_ mask: CVPixelBuffer, to image: UIImage) -> UIImage? {
        // Convert mask to alpha channel and apply to image
        // ... Core Image implementation
    }
}
```

#### 3.2 Background Mode View (Day 3-5)

```swift
// Features/Background/BackgroundView.swift
struct BackgroundView: View {
    @Bindable var appState: AppState
    @State private var selectedMode: BackgroundMode = .preservePage
    @State private var showShadow = true
    @State private var selectedBackground: BackgroundOption?
    @State private var extractedColors: [Color] = []
    @State private var processedImage: UIImage?
    @State private var isProcessing = false

    var body: some View {
        VStack(spacing: 0) {
            // Preview area
            ZStack {
                // Checkerboard for transparency
                CheckerboardView()

                if let image = processedImage {
                    Image(uiImage: image)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .padding()
                }

                if isProcessing {
                    ProgressView()
                        .scaleEffect(1.5)
                }
            }
            .frame(maxHeight: .infinity)

            // Controls
            VStack(spacing: Theme.Spacing.md) {
                // Mode selector
                Picker("Mode", selection: $selectedMode) {
                    Text("Preserve Page").tag(BackgroundMode.preservePage)
                    Text("Replace Background").tag(BackgroundMode.replaceBackground)
                }
                .pickerStyle(.segmented)
                .padding(.horizontal)

                // Background options (only for replace mode)
                if selectedMode == .replaceBackground {
                    BackgroundPicker(
                        extractedColors: extractedColors,
                        selectedBackground: $selectedBackground
                    )
                }

                // Shadow toggle
                Toggle("Drop Shadow", isOn: $showShadow)
                    .padding(.horizontal)

                // Confirm button
                Button(action: confirmBackground) {
                    Text("Continue")
                        .fontWeight(.semibold)
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color.primary)
                        .foregroundColor(.white)
                        .cornerRadius(Theme.Radius.md)
                }
                .padding()
            }
            .background(Color(.systemBackground))
        }
        .task {
            await extractColors()
            await processImage()
        }
        .onChange(of: selectedMode) { _, _ in
            Task { await processImage() }
        }
        .onChange(of: selectedBackground) { _, _ in
            Task { await processImage() }
        }
        .onChange(of: showShadow) { _, _ in
            Task { await processImage() }
        }
    }

    private func extractColors() async {
        guard let image = appState.croppedImage else { return }
        extractedColors = ImageProcessor.extractColors(from: image)
    }

    private func processImage() async {
        guard let image = appState.croppedImage else { return }
        isProcessing = true
        defer { isProcessing = false }

        switch selectedMode {
        case .preservePage:
            // Just add shadow to cropped image
            if showShadow {
                processedImage = ImageProcessor.addShadow(to: image)
            } else {
                processedImage = image
            }

        case .replaceBackground:
            // Extract subject and composite onto background
            do {
                guard let extracted = try await SubjectExtractor.extractSubject(from: image) else {
                    processedImage = image
                    return
                }

                let withBackground = compositeOnBackground(extracted, background: selectedBackground)

                if showShadow {
                    processedImage = ImageProcessor.addShadow(to: withBackground)
                } else {
                    processedImage = withBackground
                }
            } catch {
                print("Extraction failed: \(error)")
                processedImage = image
            }
        }
    }

    private func compositeOnBackground(_ subject: UIImage, background: BackgroundOption?) -> UIImage {
        // Composite subject onto selected background
        // ... implementation
    }

    private func confirmBackground() {
        appState.processedImage = processedImage
        appState.currentScreen = .tagging
    }
}
```

#### 3.3 Background Picker Component (Day 5-7)

```swift
// Features/Background/BackgroundPicker.swift
struct BackgroundPicker: View {
    let extractedColors: [Color]
    @Binding var selectedBackground: BackgroundOption?

    private let defaultColors: [Color] = [.white, .black]
    private let patterns = BackgroundAssets.patterns  // 20 patterns
    private let photos = BackgroundAssets.photos      // 20 photos

    var body: some View {
        VStack(alignment: .leading, spacing: Theme.Spacing.md) {
            // Colors row
            VStack(alignment: .leading, spacing: Theme.Spacing.sm) {
                Text("Colors")
                    .font(.caption)
                    .foregroundColor(.secondary)
                    .padding(.horizontal)

                ScrollView(.horizontal, showsIndicators: false) {
                    HStack(spacing: Theme.Spacing.sm) {
                        // Extracted colors
                        ForEach(Array(extractedColors.enumerated()), id: \.offset) { index, color in
                            ColorSwatch(color: color, isSelected: isSelected(.solid, id: "extracted-\(index)"))
                                .onTapGesture {
                                    selectBackground(.solid, color: color, id: "extracted-\(index)")
                                }
                        }

                        Divider()
                            .frame(height: 40)

                        // Default colors
                        ForEach(Array(defaultColors.enumerated()), id: \.offset) { index, color in
                            ColorSwatch(color: color, isSelected: isSelected(.solid, id: "default-\(index)"))
                                .onTapGesture {
                                    selectBackground(.solid, color: color, id: "default-\(index)")
                                }
                        }
                    }
                    .padding(.horizontal)
                }
            }

            // Patterns row
            VStack(alignment: .leading, spacing: Theme.Spacing.sm) {
                Text("Patterns")
                    .font(.caption)
                    .foregroundColor(.secondary)
                    .padding(.horizontal)

                ScrollView(.horizontal, showsIndicators: false) {
                    HStack(spacing: Theme.Spacing.sm) {
                        ForEach(patterns) { pattern in
                            BackgroundThumbnail(imageName: pattern.imageName, isSelected: selectedBackground?.id == pattern.id)
                                .onTapGesture {
                                    selectedBackground = pattern
                                }
                        }
                    }
                    .padding(.horizontal)
                }
            }

            // Photos row
            VStack(alignment: .leading, spacing: Theme.Spacing.sm) {
                Text("Photos")
                    .font(.caption)
                    .foregroundColor(.secondary)
                    .padding(.horizontal)

                ScrollView(.horizontal, showsIndicators: false) {
                    HStack(spacing: Theme.Spacing.sm) {
                        ForEach(photos) { photo in
                            BackgroundThumbnail(imageName: photo.imageName, isSelected: selectedBackground?.id == photo.id)
                                .onTapGesture {
                                    selectedBackground = photo
                                }
                        }
                    }
                    .padding(.horizontal)
                }
            }
        }
    }
}

struct ColorSwatch: View {
    let color: Color
    let isSelected: Bool

    var body: some View {
        Circle()
            .fill(color)
            .frame(width: 44, height: 44)
            .overlay(
                Circle()
                    .stroke(isSelected ? Color.primary : Color.clear, lineWidth: 3)
            )
            .shadow(radius: 1)
    }
}

struct BackgroundThumbnail: View {
    let imageName: String
    let isSelected: Bool

    var body: some View {
        Image(imageName)
            .resizable()
            .aspectRatio(contentMode: .fill)
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: Theme.Radius.sm))
            .overlay(
                RoundedRectangle(cornerRadius: Theme.Radius.sm)
                    .stroke(isSelected ? Color.primary : Color.clear, lineWidth: 3)
            )
    }
}
```

#### 3.4 Background Assets (Day 7-8)

```swift
// Resources/BackgroundAssets.swift
enum BackgroundAssets {
    static let patterns: [BackgroundOption] = [
        BackgroundOption(id: "pattern-grid", type: .pattern, color: nil, imageName: "bg_pattern_grid"),
        BackgroundOption(id: "pattern-dots", type: .pattern, color: nil, imageName: "bg_pattern_dots"),
        BackgroundOption(id: "pattern-lines", type: .pattern, color: nil, imageName: "bg_pattern_lines"),
        // ... 17 more patterns
    ]

    static let photos: [BackgroundOption] = [
        BackgroundOption(id: "photo-marble", type: .photo, color: nil, imageName: "bg_photo_marble"),
        BackgroundOption(id: "photo-concrete", type: .photo, color: nil, imageName: "bg_photo_concrete"),
        BackgroundOption(id: "photo-wood", type: .photo, color: nil, imageName: "bg_photo_wood"),
        // ... 17 more photos
    ]
}
```

**Designer deliverables needed:**
- 20 pattern images (tileable, 512x512 or 1024x1024)
- 20 photo backgrounds (high-res, 2048x2048 minimum)

### Phase 3 Deliverable
- Both background modes functional
- Subject extraction working
- Color extraction from images
- Background picker with colors/patterns/photos
- Shadow toggle working
- Processed image ready for tagging

---

## Phase 4: AI Tagging (Week 8-9)

### Goals
- Backend proxy deployed
- AI tagging generating relevant tags
- Tag editing UI working
- Tags embedded in image metadata

### Tasks

#### 4.1 API Proxy Setup (Day 1-2)

**Option A: AWS Lambda**

```javascript
// lambda/index.js
const { OpenAI } = require('openai');

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

exports.handler = async (event) => {
    // Rate limiting check
    const clientId = event.headers['x-client-id'];
    if (await isRateLimited(clientId)) {
        return { statusCode: 429, body: 'Rate limit exceeded' };
    }

    const { imageBase64 } = JSON.parse(event.body);

    const response = await openai.chat.completions.create({
        model: "gpt-4o",
        messages: [
            {
                role: "user",
                content: [
                    {
                        type: "text",
                        text: `Analyze this image from a book or magazine. Generate 5-10 tags from these categories:

STYLE: brutalist, minimalist, maximalist, editorial, swiss, bauhaus, art-deco, retro, contemporary, experimental, classic, grunge, clean, organic, geometric

TYPE: typography, photography, illustration, layout, poster, cover, spread, infographic, pattern, logo, pull-quote

COLOR: black-and-white, monochrome, vibrant, muted, pastel, high-contrast, duotone, warm, cool, earth-tones

FORMAT: full-page, spread, detail, vertical, horizontal, square, grid, full-bleed

SUBJECT: architecture, fashion, product, portrait, landscape, abstract, food, interiors, nature, urban, literature, poetry

Return ONLY a JSON array of lowercase tag strings. No explanation.`
                    },
                    {
                        type: "image_url",
                        image_url: { url: `data:image/jpeg;base64,${imageBase64}` }
                    }
                ]
            }
        ],
        max_tokens: 150
    });

    const tags = JSON.parse(response.choices[0].message.content);

    // Log usage for billing
    await logUsage(clientId, 1);

    return {
        statusCode: 200,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ tags })
    };
};
```

**Option B: Cloudflare Workers (simpler)**

```javascript
// worker.js
export default {
    async fetch(request, env) {
        if (request.method !== 'POST') {
            return new Response('Method not allowed', { status: 405 });
        }

        const { imageBase64 } = await request.json();

        const response = await fetch('https://api.openai.com/v1/chat/completions', {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${env.OPENAI_API_KEY}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                model: 'gpt-4o',
                messages: [/* same as above */],
                max_tokens: 150
            })
        });

        const data = await response.json();
        const tags = JSON.parse(data.choices[0].message.content);

        return new Response(JSON.stringify({ tags }), {
            headers: { 'Content-Type': 'application/json' }
        });
    }
};
```

#### 4.2 Tagging Service (Day 2-3)

```swift
// Services/TaggingService.swift
class TaggingService {
    private let endpoint = "https://your-api-endpoint.com/analyze"

    func generateTags(for image: UIImage) async throws -> [String] {
        guard let imageData = image.jpegData(compressionQuality: 0.8) else {
            throw TaggingError.invalidImage
        }

        let base64 = imageData.base64EncodedString()

        var request = URLRequest(url: URL(string: endpoint)!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue(deviceId, forHTTPHeaderField: "X-Client-ID")
        request.httpBody = try JSONEncoder().encode(["imageBase64": base64])

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw TaggingError.invalidResponse
        }

        switch httpResponse.statusCode {
        case 200:
            let result = try JSONDecoder().decode(TagResponse.self, from: data)
            return result.tags
        case 429:
            throw TaggingError.rateLimited
        default:
            throw TaggingError.serverError(httpResponse.statusCode)
        }
    }

    private var deviceId: String {
        // Use identifierForVendor or generate UUID stored in keychain
        UIDevice.current.identifierForVendor?.uuidString ?? UUID().uuidString
    }
}

struct TagResponse: Codable {
    let tags: [String]
}

enum TaggingError: Error {
    case invalidImage
    case invalidResponse
    case rateLimited
    case serverError(Int)
}
```

#### 4.3 Tagging View (Day 3-5)

```swift
// Features/Tagging/TaggingView.swift
struct TaggingView: View {
    @Bindable var appState: AppState
    @State private var suggestedTags: [String] = []
    @State private var selectedTags: Set<String> = []
    @State private var customTagInput = ""
    @State private var isLoading = true
    @State private var error: Error?

    private let taggingService = TaggingService()

    var body: some View {
        VStack(spacing: 0) {
            // Preview
            if let image = appState.processedImage {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fit)
                    .frame(maxHeight: 300)
                    .padding()
            }

            Divider()

            // Tags section
            ScrollView {
                VStack(alignment: .leading, spacing: Theme.Spacing.md) {
                    // Loading state
                    if isLoading {
                        HStack {
                            ProgressView()
                            Text("Analyzing image...")
                                .foregroundColor(.secondary)
                        }
                        .frame(maxWidth: .infinity)
                        .padding()
                    }

                    // Suggested tags
                    if !suggestedTags.isEmpty {
                        Text("Suggested Tags")
                            .font(.headline)

                        FlowLayout(spacing: Theme.Spacing.sm) {
                            ForEach(suggestedTags, id: \.self) { tag in
                                TagChip(
                                    tag: tag,
                                    isSelected: selectedTags.contains(tag),
                                    onTap: { toggleTag(tag) },
                                    onRemove: nil
                                )
                            }
                        }
                    }

                    // Custom tag input
                    Text("Add Custom Tags")
                        .font(.headline)

                    HStack {
                        TextField("Type a tag...", text: $customTagInput)
                            .textFieldStyle(.roundedBorder)
                            .autocapitalization(.none)
                            .onSubmit { addCustomTag() }

                        Button(action: addCustomTag) {
                            Image(systemName: "plus.circle.fill")
                                .font(.title2)
                        }
                        .disabled(customTagInput.isEmpty)
                    }

                    // Selected tags
                    if !selectedTags.isEmpty {
                        Text("Selected (\(selectedTags.count))")
                            .font(.headline)

                        FlowLayout(spacing: Theme.Spacing.sm) {
                            ForEach(Array(selectedTags).sorted(), id: \.self) { tag in
                                TagChip(
                                    tag: tag,
                                    isSelected: true,
                                    onTap: { toggleTag(tag) },
                                    onRemove: { removeTag(tag) }
                                )
                            }
                        }
                    }
                }
                .padding()
            }

            // Confirm button
            Button(action: confirmTags) {
                Text("Continue")
                    .fontWeight(.semibold)
                    .frame(maxWidth: .infinity)
                    .padding()
                    .background(selectedTags.isEmpty ? Color.gray : Color.primary)
                    .foregroundColor(.white)
                    .cornerRadius(Theme.Radius.md)
            }
            .disabled(selectedTags.isEmpty)
            .padding()
        }
        .task {
            await loadTags()
        }
    }

    private func loadTags() async {
        guard let image = appState.processedImage else { return }

        do {
            suggestedTags = try await taggingService.generateTags(for: image)
            // Auto-select all suggested tags
            selectedTags = Set(suggestedTags)
        } catch {
            self.error = error
            // Fallback: allow manual tagging only
        }

        isLoading = false
    }

    private func toggleTag(_ tag: String) {
        if selectedTags.contains(tag) {
            selectedTags.remove(tag)
        } else {
            selectedTags.insert(tag)
        }
    }

    private func addCustomTag() {
        let tag = customTagInput.lowercased().trimmingCharacters(in: .whitespaces)
        guard !tag.isEmpty else { return }
        selectedTags.insert(tag)
        if !suggestedTags.contains(tag) {
            suggestedTags.append(tag)
        }
        customTagInput = ""
    }

    private func removeTag(_ tag: String) {
        selectedTags.remove(tag)
    }

    private func confirmTags() {
        appState.selectedTags = Array(selectedTags)
        appState.currentScreen = .export
    }
}

struct TagChip: View {
    let tag: String
    let isSelected: Bool
    let onTap: () -> Void
    let onRemove: (() -> Void)?

    var body: some View {
        HStack(spacing: 4) {
            Text(tag)
                .font(.subheadline)

            if let onRemove, isSelected {
                Button(action: onRemove) {
                    Image(systemName: "xmark.circle.fill")
                        .font(.caption)
                }
            }
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 6)
        .background(isSelected ? Color.primary.opacity(0.15) : Color(.systemGray5))
        .foregroundColor(isSelected ? .primary : .secondary)
        .cornerRadius(Theme.Radius.lg)
        .onTapGesture(perform: onTap)
    }
}
```

#### 4.4 Metadata Embedding (Day 5-7)

```swift
// Services/MetadataService.swift
import ImageIO
import UniformTypeIdentifiers

class MetadataService {

    /// Embed IPTC keywords into image
    static func embedTags(_ tags: [String], in image: UIImage) -> Data? {
        guard let imageData = image.jpegData(compressionQuality: 0.95),
              let source = CGImageSourceCreateWithData(imageData as CFData, nil),
              let uti = CGImageSourceGetType(source) else {
            return nil
        }

        // Get existing metadata
        var metadata = CGImageSourceCopyPropertiesAtIndex(source, 0, nil) as? [String: Any] ?? [:]

        // Add IPTC keywords
        var iptc = metadata[kCGImagePropertyIPTCDictionary as String] as? [String: Any] ?? [:]
        iptc[kCGImagePropertyIPTCKeywords as String] = tags
        metadata[kCGImagePropertyIPTCDictionary as String] = iptc

        // Also add to XMP for broader compatibility
        var xmp = metadata[kCGImagePropertyExifDictionary as String] as? [String: Any] ?? [:]
        xmp["UserComment"] = tags.joined(separator: ", ")
        metadata[kCGImagePropertyExifDictionary as String] = xmp

        // Create new image with metadata
        let outputData = NSMutableData()
        guard let destination = CGImageDestinationCreateWithData(outputData, uti, 1, nil) else {
            return nil
        }

        CGImageDestinationAddImageFromSource(destination, source, 0, metadata as CFDictionary)

        guard CGImageDestinationFinalize(destination) else {
            return nil
        }

        return outputData as Data
    }

    /// Generate filename from tags
    static func generateFilename(from tags: [String]) -> String {
        let prefix = "printshot"
        let timestamp = Int(Date().timeIntervalSince1970)
        let tagString = tags.prefix(3).joined(separator: "-")
        return "\(prefix)_\(tagString)_\(timestamp).jpg"
    }
}
```

### Phase 4 Deliverable
- API proxy deployed and working
- AI generates relevant tags from images
- Tag editing UI complete
- Tags embedded in IPTC metadata
- Filename generated from tags

---

## Phase 5: Export & Polish (Week 10-11)

### Goals
- Share sheet integration
- Save to camera roll
- Settings screen
- Performance optimization
- Bug fixes

### Tasks

#### 5.1 Export View (Day 1-2)

```swift
// Features/Export/ExportView.swift
struct ExportView: View {
    @Bindable var appState: AppState
    @State private var finalImageData: Data?
    @State private var showShareSheet = false
    @State private var showSaveSuccess = false

    var body: some View {
        VStack(spacing: Theme.Spacing.lg) {
            // Preview
            if let image = appState.processedImage {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fit)
                    .frame(maxHeight: 400)
                    .padding()
            }

            // Tags display
            if !appState.selectedTags.isEmpty {
                FlowLayout(spacing: Theme.Spacing.xs) {
                    ForEach(appState.selectedTags, id: \.self) { tag in
                        Text(tag)
                            .font(.caption)
                            .padding(.horizontal, 8)
                            .padding(.vertical, 4)
                            .background(Color(.systemGray5))
                            .cornerRadius(Theme.Radius.sm)
                    }
                }
                .padding(.horizontal)
            }

            Spacer()

            // Action buttons
            VStack(spacing: Theme.Spacing.md) {
                // Share button
                Button(action: { showShareSheet = true }) {
                    Label("Share", systemImage: "square.and.arrow.up")
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color.primary)
                        .foregroundColor(.white)
                        .cornerRadius(Theme.Radius.md)
                }

                // Save button
                Button(action: saveToPhotos) {
                    Label("Save to Photos", systemImage: "photo.on.rectangle")
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color(.systemGray5))
                        .foregroundColor(.primary)
                        .cornerRadius(Theme.Radius.md)
                }

                // New capture button
                Button(action: newCapture) {
                    Text("New Capture")
                        .foregroundColor(.secondary)
                }
            }
            .padding()
        }
        .sheet(isPresented: $showShareSheet) {
            if let data = finalImageData {
                ShareSheet(items: [data])
            }
        }
        .alert("Saved!", isPresented: $showSaveSuccess) {
            Button("OK", role: .cancel) { }
        }
        .task {
            prepareExport()
        }
    }

    private func prepareExport() {
        guard let image = appState.processedImage else { return }
        finalImageData = MetadataService.embedTags(appState.selectedTags, in: image)
    }

    private func saveToPhotos() {
        guard let image = appState.processedImage else { return }
        UIImageWriteToSavedPhotosAlbum(image, nil, nil, nil)
        showSaveSuccess = true
    }

    private func newCapture() {
        appState.reset()
    }
}

struct ShareSheet: UIViewControllerRepresentable {
    let items: [Any]

    func makeUIViewController(context: Context) -> UIActivityViewController {
        UIActivityViewController(activityItems: items, applicationActivities: nil)
    }

    func updateUIViewController(_ uiViewController: UIActivityViewController, context: Context) {}
}
```

#### 5.2 Settings View (Day 2-3)

```swift
// Features/Settings/SettingsView.swift
struct SettingsView: View {
    @AppStorage("defaultShadow") private var defaultShadow = true
    @AppStorage("defaultBackgroundMode") private var defaultBackgroundMode = "preservePage"
    @AppStorage("imageQuality") private var imageQuality = 0.95

    var body: some View {
        NavigationStack {
            Form {
                Section("Defaults") {
                    Toggle("Drop Shadow", isOn: $defaultShadow)

                    Picker("Background Mode", selection: $defaultBackgroundMode) {
                        Text("Preserve Page").tag("preservePage")
                        Text("Replace Background").tag("replaceBackground")
                    }
                }

                Section("Export") {
                    Picker("Image Quality", selection: $imageQuality) {
                        Text("Maximum").tag(1.0)
                        Text("High").tag(0.95)
                        Text("Medium").tag(0.8)
                    }
                }

                Section("About") {
                    HStack {
                        Text("Version")
                        Spacer()
                        Text("1.0.0")
                            .foregroundColor(.secondary)
                    }

                    Link("Privacy Policy", destination: URL(string: "https://printshot.app/privacy")!)
                    Link("Terms of Service", destination: URL(string: "https://printshot.app/terms")!)
                }

                Section {
                    Link("Send Feedback", destination: URL(string: "mailto:hello@printshot.app")!)
                    Link("Rate on App Store", destination: URL(string: "https://apps.apple.com/app/printshot/id123456")!)
                }
            }
            .navigationTitle("Settings")
        }
    }
}
```

#### 5.3 Performance Optimization (Day 4-6)

```swift
// Optimization checklist:

// 1. Image resizing before AI analysis (reduce API costs and latency)
extension UIImage {
    func resized(maxDimension: CGFloat) -> UIImage {
        let scale = min(maxDimension / size.width, maxDimension / size.height)
        if scale >= 1 { return self }

        let newSize = CGSize(width: size.width * scale, height: size.height * scale)
        let renderer = UIGraphicsImageRenderer(size: newSize)
        return renderer.image { _ in
            draw(in: CGRect(origin: .zero, size: newSize))
        }
    }
}

// 2. Background processing for AI tagging
// Start tag generation while user is adjusting background
// Show loading state but don't block UI

// 3. Lazy loading for background assets
// Only load pattern/photo thumbnails initially
// Load full resolution when selected

// 4. Memory management
// Release captured image after cropping
// Release cropped image after processing
// Use autoreleasepool for image processing loops

// 5. Caching
// Cache extracted colors
// Cache processed preview (invalidate on setting change)
```

#### 5.4 Error Handling & Edge Cases (Day 6-8)

```swift
// Error states to handle:

// 1. Camera permission denied
struct CameraPermissionDeniedView: View {
    var body: some View {
        VStack(spacing: Theme.Spacing.lg) {
            Image(systemName: "camera.fill")
                .font(.system(size: 60))
                .foregroundColor(.secondary)

            Text("Camera Access Required")
                .font(.title2)
                .fontWeight(.semibold)

            Text("PrintShot needs camera access to capture pages. Please enable it in Settings.")
                .multilineTextAlignment(.center)
                .foregroundColor(.secondary)

            Button("Open Settings") {
                UIApplication.shared.open(URL(string: UIApplication.openSettingsURLString)!)
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}

// 2. AI tagging failure
// - Show error message
// - Allow manual tagging only
// - Retry button

// 3. Background removal failure
// - Fall back to original cropped image
// - Show subtle toast: "Couldn't remove background"

// 4. Low memory warning
// - Reduce image quality temporarily
// - Show warning if needed

// 5. Network unavailable
// - Detect before tagging screen
// - Skip AI, go straight to manual tagging
// - Or show "Tagging requires internet" message
```

### Phase 5 Deliverable
- Share sheet working with all destinations
- Save to photos working
- Settings screen complete
- App performs well (< 3s processing time)
- Error states handled gracefully

---

## Phase 6: Launch Prep (Week 12)

### Tasks

#### 6.1 App Store Assets (Day 1-2)

**Required:**
- [ ] App icon (1024x1024)
- [ ] Screenshots (6.7", 6.5", 5.5" iPhones)
- [ ] App preview video (optional but recommended, 15-30s)
- [ ] App description (4000 char max)
- [ ] Keywords (100 char max)
- [ ] What's New text
- [ ] Privacy policy URL
- [ ] Support URL

**Screenshot suggestions:**
1. Camera with capture button (hero shot)
2. Crop interface in action
3. "Preserve Page" mode result
4. "Replace Background" mode with color picker
5. AI tags screen
6. Share/export screen

#### 6.2 App Description Draft

```
PrintShot — Your reading life, made searchable and shareable.

Capture what you see in books and magazines. Preserve the texture of the page or extract elements onto clean backgrounds. Auto-tag for searchability. Share to Instagram, Are.na, Obsidian, or anywhere.

CAPTURE IN SECONDS
• Instant camera launch
• Manual crop with precision handles
• Include or exclude your hands — you decide

TWO BACKGROUND MODES
• Preserve Page: Keep the paper texture, typography, and character
• Replace Background: Extract content onto solid colors, patterns, or photos

SMART TAGGING
• AI-powered tag suggestions
• Add your own tags
• Tags embedded in image metadata
• Searchable in Finder, Photos, and PKM tools

SHARE ANYWHERE
• Native share sheet
• Save to Camera Roll
• Perfect for Instagram, Are.na, Obsidian, Notes

Built for people who read physical things.

---
Privacy: Images are processed on-device. Only the final image is sent for AI tagging. No accounts required. No data stored on our servers.
```

#### 6.3 Beta Testing (Day 2-4)

```
TestFlight rollout:
- Internal team: Day 2
- Seed creators (20-30 people): Day 3
- Open beta (optional): Day 4

Feedback to collect:
- Capture flow timing (is it under 15 seconds?)
- Background removal quality
- Tag relevance
- Any crashes or errors
- Feature requests for v1.1
```

#### 6.4 Final QA Checklist (Day 4-5)

```
Functional:
[ ] Camera captures correctly
[ ] Flash works
[ ] Crop handles drag smoothly
[ ] Preserve Page mode works
[ ] Replace Background mode works
[ ] All 20 patterns load
[ ] All 20 photos load
[ ] Color extraction works
[ ] Shadow toggle works
[ ] AI tagging returns results
[ ] Manual tag add works
[ ] Tag removal works
[ ] Metadata embedded correctly
[ ] Share sheet opens
[ ] Save to Photos works
[ ] Settings persist
[ ] New capture resets state

Edge cases:
[ ] Deny camera permission → proper error
[ ] No network → graceful fallback
[ ] Very large image → doesn't crash
[ ] Very small crop → handled
[ ] Rapid screen transitions → no crashes
[ ] Background app → resume correctly

Performance:
[ ] Camera launch < 1s
[ ] Capture to crop < 0.5s
[ ] Background processing < 2s
[ ] AI tagging < 3s
[ ] Export < 1s
[ ] Memory stays under 200MB
```

#### 6.5 App Store Submission (Day 5)

```
Submission checklist:
[ ] Archive build in Xcode
[ ] Upload to App Store Connect
[ ] Fill all metadata
[ ] Upload screenshots
[ ] Set pricing ($5.99)
[ ] Set availability (all countries)
[ ] Submit for review

Expected review time: 24-48 hours
```

### Phase 6 Deliverable
- App submitted to App Store
- Marketing assets ready
- Beta feedback incorporated
- Ready for launch

---

## Risk Mitigation Summary

| Risk | Mitigation | Owner |
|------|------------|-------|
| Background removal quality | Test extensively, have "original" fallback | Dev |
| AI tagging latency | Background processing, optimistic UI | Dev |
| AI tagging cost | Rate limiting, usage caps, monitor closely | Dev/Ops |
| App Store rejection | Follow guidelines, honest description | Dev |
| Designer delays | Have fallback placeholder assets | PM |
| Scope creep | Strict MVP boundary, defer to v1.1 | PM |

---

## Success Criteria for Launch

| Metric | Target |
|--------|--------|
| Crash-free rate | > 99% |
| Average capture time | < 15 seconds |
| App Store rating | > 4.5 |
| Week 1 downloads | > 500 |

---

## Post-Launch Priorities (v1.1)

1. In-app library with tag search
2. Carousel export for Books Wrapped
3. Additional background packs
4. On-device tagging option
5. Performance improvements based on analytics

---

*Implementation plan prepared for PrintShot iOS app. Timeline: 12 weeks.*
