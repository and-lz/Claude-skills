# Media Playback — Complete Reference

## Table of contents
1. AVPlayerViewController on tvOS
2. Transport bar customization
3. Info panel and metadata
4. Interstitials (ad breaks)
5. Picture in Picture
6. Background audio
7. Passthrough audio (tvOS 26)
8. Custom player controls
9. VLCKit on tvOS
10. Rules and anti-patterns

---

## 1. AVPlayerViewController on tvOS

`AVPlayerViewController` is the standard, recommended player for tvOS. It provides the full native transport experience: scrubbing, chapter support, audio/subtitle selection, and Siri integration.

```swift
import AVKit
import SwiftUI

struct TVPlayerView: UIViewControllerRepresentable {
    let url: URL
    let title: String
    let subtitle: String

    func makeUIViewController(context: Context) -> AVPlayerViewController {
        let controller = AVPlayerViewController()
        let player = AVPlayer(url: url)
        controller.player = player

        // Metadata for transport bar
        let item = player.currentItem
        let titleItem = AVMutableMetadataItem()
        titleItem.identifier = .commonIdentifierTitle
        titleItem.value = title as NSString

        let subtitleItem = AVMutableMetadataItem()
        subtitleItem.identifier = .commonIdentifierDescription
        subtitleItem.value = subtitle as NSString

        item?.externalMetadata = [titleItem, subtitleItem]

        // Configure behavior
        controller.allowsPictureInPicturePlayback = true
        controller.requiresLinearPlayback = false
        controller.skippingBehavior = .default  // or .skipItem

        // Start playback
        player.play()

        return controller
    }

    func updateUIViewController(_ controller: AVPlayerViewController, context: Context) { }
}

// Usage
struct ContentView: View {
    @State private var showPlayer = false

    var body: some View {
        Button("Play Movie") {
            showPlayer = true
        }
        .fullScreenCover(isPresented: $showPlayer) {
            TVPlayerView(
                url: streamURL,
                title: "Movie Title",
                subtitle: "2025 · Action · 2h 15m"
            )
            .ignoresSafeArea()
        }
    }
}
```

## 2. Transport bar customization

```swift
func makeUIViewController(context: Context) -> AVPlayerViewController {
    let controller = AVPlayerViewController()

    // Transport bar title view
    controller.transportBarIncludesTitleView = true

    // Custom menu items in transport bar
    controller.transportBarCustomMenuItems = [
        UIAction(title: "Audio & Subtitles", image: UIImage(systemName: "text.bubble")) { _ in
            showAudioSubtitlePicker()
        },
        UIAction(title: "Playback Speed", image: UIImage(systemName: "gauge.with.needle")) { _ in
            showSpeedPicker()
        },
        UIAction(title: "Quality", image: UIImage(systemName: "slider.horizontal.3")) { _ in
            showQualityPicker()
        }
    ]

    // Contextual actions (appear above transport bar)
    controller.contextualActions = [
        UIAction(title: "Add to Watchlist", image: UIImage(systemName: "plus")) { _ in
            addToWatchlist()
        },
        UIAction(title: "Like", image: UIImage(systemName: "hand.thumbsup")) { _ in
            likeContent()
        }
    ]

    // Skip behavior
    controller.skippingBehavior = .default        // 10s forward/backward
    // controller.skippingBehavior = .skipItem     // Skip to next/previous item

    return controller
}
```

## 3. Info panel and metadata

The info panel appears when the user swipes down during playback. Customize it with metadata and additional view controllers.

```swift
func configureInfoPanel(_ controller: AVPlayerViewController) {
    // Custom info tabs
    let castVC = makeCastViewController()    // Cast & crew grid
    let episodesVC = makeEpisodesViewController()  // Episode list
    let relatedVC = makeRelatedViewController()    // Related content

    controller.customInfoViewControllers = [castVC, episodesVC, relatedVC]

    // Custom overlay (appears during playback)
    let overlayVC = makeOverlayViewController()  // Skip intro button, etc.
    controller.customOverlayViewController = overlayVC

    // Content tab label customization
    controller.infoViewActions = [
        UIAction(title: "From Beginning") { _ in
            controller.player?.seek(to: .zero)
        }
    ]
}

// Metadata for info panel display
func setPlayerMetadata(_ item: AVPlayerItem, movie: Movie) {
    var metadata: [AVMutableMetadataItem] = []

    // Title
    let title = AVMutableMetadataItem()
    title.identifier = .commonIdentifierTitle
    title.value = movie.title as NSString
    metadata.append(title)

    // Description
    let desc = AVMutableMetadataItem()
    desc.identifier = .commonIdentifierDescription
    desc.value = movie.overview as NSString
    metadata.append(desc)

    // Artwork
    if let imageData = try? Data(contentsOf: movie.posterURL) {
        let artwork = AVMutableMetadataItem()
        artwork.identifier = .commonIdentifierArtwork
        artwork.value = imageData as NSData
        artwork.dataType = kCMMetadataBaseDataType_JPEG as String
        metadata.append(artwork)
    }

    // Genre
    let genre = AVMutableMetadataItem()
    genre.identifier = .commonIdentifierType
    genre.value = movie.genres.joined(separator: ", ") as NSString
    metadata.append(genre)

    // Year
    let year = AVMutableMetadataItem()
    year.identifier = .commonIdentifierCreationDate
    year.value = "\(movie.year)" as NSString
    metadata.append(year)

    item.externalMetadata = metadata
}
```

## 4. Interstitials (ad breaks)

```swift
// Mark time ranges as interstitials (shown as dots on timeline)
func addInterstitials(_ playerItem: AVPlayerItem) {
    let preRoll = AVInterstitialTimeRange(
        timeRange: CMTimeRange(
            start: .zero,
            duration: CMTime(seconds: 30, preferredTimescale: 600)
        )
    )

    let midRoll = AVInterstitialTimeRange(
        timeRange: CMTimeRange(
            start: CMTime(seconds: 1800, preferredTimescale: 600),  // 30 minutes in
            duration: CMTime(seconds: 60, preferredTimescale: 600)
        )
    )

    playerItem.interstitialTimeRanges = [preRoll, midRoll]
    // Timeline shows dots at interstitial positions
    // User cannot scrub through interstitials if requiresLinearPlayback is set
}
```

## 5. Picture in Picture

PiP is supported on tvOS 14+. The PiP window appears in a corner while the user browses other content.

```swift
// PiP is automatic when using AVPlayerViewController
let controller = AVPlayerViewController()
controller.allowsPictureInPicturePlayback = true

// PiP delegate
class PlayerCoordinator: NSObject, AVPlayerViewControllerDelegate {
    func playerViewControllerWillStartPictureInPicture(_ playerViewController: AVPlayerViewController) {
        // Save playback state
    }

    func playerViewControllerDidStopPictureInPicture(_ playerViewController: AVPlayerViewController) {
        // Resume inline playback
    }

    func playerViewController(
        _ playerViewController: AVPlayerViewController,
        restoreUserInterfaceForPictureInPictureStopWithCompletionHandler completionHandler: @escaping (Bool) -> Void
    ) {
        // Restore full-screen player UI
        completionHandler(true)
    }
}
```

## 6. Background audio

```swift
import AVFoundation

// Set audio session for background playback
func configureAudioSession() {
    do {
        try AVAudioSession.sharedInstance().setCategory(
            .playback,
            mode: .moviePlayback,  // or .default for music
            options: []
        )
        try AVAudioSession.sharedInstance().setActive(true)
    } catch {
        print("Audio session error: \(error)")
    }
}

// Remote control events for background
func setupNowPlaying(_ item: MediaItem) {
    var info = [String: Any]()
    info[MPMediaItemPropertyTitle] = item.title
    info[MPMediaItemPropertyArtist] = item.artist
    info[MPNowPlayingInfoPropertyPlaybackRate] = 1.0
    info[MPMediaItemPropertyPlaybackDuration] = item.duration
    info[MPNowPlayingInfoPropertyElapsedPlaybackTime] = player.currentTime().seconds

    MPNowPlayingInfoCenter.default().nowPlayingInfo = info
}
```

## 7. Passthrough audio (tvOS 26)

New in tvOS 26: `AVAudioContentSource.passthrough` enables streaming high-quality audio formats directly to external audio processors.

```swift
// Supported passthrough formats:
// - Dolby TrueHD
// - DTS / DTS-HD MA
// - Auro3D
// - AC-3 / E-AC-3 (already supported in prior versions)

// Configure for passthrough
let audioSession = AVAudioSession.sharedInstance()
try audioSession.setCategory(
    .playback,
    mode: .moviePlayback,
    routeSharingPolicy: .longFormAudio
)

// Check if passthrough is available for current route
if let route = audioSession.currentRoute.outputs.first {
    // Check if HDMI/eARC supports the format
    print("Output: \(route.portType), channels: \(route.channels?.count ?? 0)")
}
```

## 8. Custom player controls

For cases where `AVPlayerViewController` doesn't meet your needs (e.g., VLCKit, custom streaming):

```swift
struct CustomPlayerOverlay: View {
    @Binding var isPlaying: Bool
    @Binding var progress: Double
    @Binding var showControls: Bool
    let duration: TimeInterval

    var body: some View {
        ZStack {
            // Tap anywhere to toggle controls
            Color.clear
                .focusable()

            if showControls {
                VStack {
                    Spacer()

                    // Transport controls
                    HStack(spacing: 60) {
                        Button {
                            skipBackward()
                        } label: {
                            Image(systemName: "gobackward.15")
                                .font(.system(size: 44))
                        }

                        Button {
                            isPlaying.toggle()
                        } label: {
                            Image(systemName: isPlaying ? "pause" : "play.fill")
                                .font(.system(size: 64))
                        }

                        Button {
                            skipForward()
                        } label: {
                            Image(systemName: "goforward.15")
                                .font(.system(size: 44))
                        }
                    }

                    // Progress bar
                    HStack {
                        Text(formatTime(progress * duration))
                            .font(.caption)
                            .monospacedDigit()

                        ProgressView(value: progress)
                            .tint(.white)

                        Text(formatTime(duration))
                            .font(.caption)
                            .monospacedDigit()
                    }
                    .padding(.horizontal, 80)
                    .padding(.bottom, 40)
                }
                .transition(.opacity)
            }
        }
        .onPlayPauseCommand {
            isPlaying.toggle()
        }
        .onExitCommand {
            if showControls {
                showControls = false
            }
        }
        .onMoveCommand { direction in
            switch direction {
            case .left: skipBackward()
            case .right: skipForward()
            case .up, .down: showControls = true
            @unknown default: break
            }
        }
    }
}
```

## 9. VLCKit on tvOS

For codecs not supported by AVFoundation (e.g., MKV containers, certain subtitle formats):

```swift
// TVVLCKit is the tvOS variant of MobileVLCKit
// Import: import TVVLCKit

// VLCMediaPlayer wrapped in UIViewControllerRepresentable
class TVVLCPlayerController: UIViewController {
    let mediaPlayer = VLCMediaPlayer()

    override func viewDidLoad() {
        super.viewDidLoad()

        // Set drawable view
        mediaPlayer.drawable = view

        // Configure media
        let media = VLCMedia(url: streamURL)
        media.addOptions([
            "network-caching": 3000,
            "clock-jitter": 0,
        ])
        mediaPlayer.media = media
    }

    func play() { mediaPlayer.play() }
    func pause() { mediaPlayer.pause() }
    func seek(to position: Float) { mediaPlayer.position = position }
}
```

## 10. Rules and anti-patterns

### DO:
- Use `AVPlayerViewController` as the default player — it provides full native transport
- Set metadata (title, artwork, description) for the info panel
- Support PiP for video content
- Configure audio session for background playback
- Handle `.onPlayPauseCommand` globally in media views
- Use `.fullScreenCover` to present players
- Set `externalMetadata` for Siri "What did they say?" support

### DON'T:
- Don't build a completely custom player when `AVPlayerViewController` works
- Don't forget to set the audio session category
- Don't block the main thread during media loading
- Don't ignore transport bar customization — it's how users discover features
- Don't skip metadata — the info panel looks empty without it
- Don't present the player in a sheet (use `.fullScreenCover`)
- Don't forget to handle the Menu button during playback (dismiss player)
