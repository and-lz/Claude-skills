# Parallax Effects — Complete Reference

## Table of contents
1. What is TV parallax
2. Layered images (.lsr)
3. TVPosterView (UIKit)
4. SwiftUI custom parallax
5. Asset creation guidelines
6. Focus-driven motion
7. Rules and anti-patterns

---

## 1. What is TV parallax

Parallax is the signature visual effect on Apple TV. When a user moves their finger on the Siri Remote's touch surface while focused on an element, the layers of the element shift independently — creating a 3D depth illusion. Combined with focus scaling and shadow changes, it makes the TV UI feel alive and spatial.

**Three components of the parallax effect:**
1. **Layer separation** — foreground shifts more than background
2. **Focus scale** — element grows ~1.05-1.10x when focused
3. **Shadow projection** — shadow deepens and shifts with the tilt

## 2. Layered images (.lsr)

Apple's recommended approach: **Layered Source Resource** (.lsr) files in Xcode Asset Catalog. The system handles all parallax math automatically.

### Creating layered images in Xcode

1. In Asset Catalog, click **+** → **Image Stack** (or "TV Image Stack")
2. Add 2-5 layers:
   - **Back layer** — background (sky, solid color, blurred scene)
   - **Middle layer(s)** — supplementary elements (characters, objects)
   - **Front layer** — foreground text, logo, or key element
3. Each layer is a separate image with transparency (PNG)
4. Export as `.lsr` file

### Layer specifications

| Layer | Parallax shift | Image requirements |
|-------|---------------|-------------------|
| Background | ±5pt | Extend 20-40pt beyond frame to prevent edge reveal |
| Middle | ±10pt | Extend 15-30pt beyond frame |
| Foreground | ±15pt | Can be within frame (text/logos) |

```swift
// Usage in SwiftUI — just reference the asset name
Image("MoviePoster_Layered")  // .lsr asset in catalog
    .frame(width: 250, height: 375)
    // System automatically applies parallax on focus
```

### Using Parallax Previewer app

Apple provides **Parallax Previewer** (available on Mac App Store or in Xcode developer tools):
1. Open your layered image
2. Preview the parallax effect with mouse movement
3. Adjust layer positions and sizes
4. Export as `.lsr`

## 3. TVPosterView (UIKit)

`TVPosterView` from TVUIKit provides a ready-made poster component with parallax, title, and subtitle.

```swift
import TVUIKit

class PosterCollectionCell: UICollectionViewCell {
    let posterView = TVPosterView()

    func configure(with item: ContentItem) {
        // Set poster image (layered or flat)
        posterView.image = UIImage(named: item.posterImageName)

        // Title and subtitle
        posterView.title = item.title
        posterView.subtitle = "\(item.year) · \(item.genre)"

        // The system automatically applies:
        // - Focus scale (1.05x)
        // - Parallax (if layered image)
        // - Shadow
        // - Title visibility (appears on focus)

        contentView.addSubview(posterView)
        posterView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            posterView.topAnchor.constraint(equalTo: contentView.topAnchor),
            posterView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor),
            posterView.trailingAnchor.constraint(equalTo: contentView.trailingAnchor),
            posterView.bottomAnchor.constraint(equalTo: contentView.bottomAnchor),
        ])
    }
}
```

### TVPosterView properties

```swift
posterView.image = UIImage(named: "poster")  // Required
posterView.title = "Movie Title"              // Shown below poster on focus
posterView.subtitle = "2025"                  // Shown below title on focus
posterView.imageView.adjustsImageWhenAncestorFocused = true  // Enable parallax
posterView.headerContentView = customHeaderView  // Optional top overlay
posterView.footerContentView = customFooterView  // Optional bottom overlay
```

## 4. SwiftUI custom parallax

For cases where you don't have layered image assets, build parallax in SwiftUI:

### Basic focus-scale parallax

```swift
struct ParallaxPoster: View {
    let item: ContentItem
    @Environment(\.isFocused) var isFocused

    var body: some View {
        AsyncImage(url: item.posterURL) { image in
            image.resizable().aspectRatio(2/3, contentMode: .fill)
        } placeholder: {
            RoundedRectangle(cornerRadius: 12)
                .fill(.quaternary)
        }
        .frame(width: 250, height: 375)
        .clipShape(RoundedRectangle(cornerRadius: 12))
        .scaleEffect(isFocused ? 1.08 : 1.0)
        .shadow(
            color: .black.opacity(isFocused ? 0.5 : 0.15),
            radius: isFocused ? 25 : 8,
            y: isFocused ? 15 : 3
        )
        .animation(.spring(response: 0.35, dampingFraction: 0.7), value: isFocused)
        .focusable()
    }
}
```

### Multi-layer parallax with focus

```swift
struct LayeredParallaxCard: View {
    let item: ContentItem
    @Environment(\.isFocused) var isFocused

    var body: some View {
        ZStack {
            // Background layer — shifts less
            AsyncImage(url: item.backdropURL) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                Rectangle().fill(.quaternary)
            }
            .offset(y: isFocused ? -5 : 0)
            .scaleEffect(isFocused ? 1.12 : 1.02)  // Slightly larger to prevent edge reveal

            // Gradient overlay
            LinearGradient(
                colors: [.clear, .black.opacity(0.7)],
                startPoint: .center,
                endPoint: .bottom
            )

            // Foreground layer — shifts more
            VStack {
                Spacer()
                Text(item.title)
                    .font(.headline)
                    .foregroundStyle(.white)
                    .shadow(radius: 4)
                    .offset(y: isFocused ? 5 : 0)
            }
            .padding()
        }
        .frame(width: 350, height: 200)
        .clipShape(RoundedRectangle(cornerRadius: 16))
        .scaleEffect(isFocused ? 1.05 : 1.0)
        .shadow(
            color: .black.opacity(isFocused ? 0.5 : 0.2),
            radius: isFocused ? 20 : 5,
            y: isFocused ? 12 : 3
        )
        .animation(.spring(response: 0.4, dampingFraction: 0.65), value: isFocused)
        .focusable()
    }
}
```

### Hero banner with parallax text

```swift
struct HeroParallaxBanner: View {
    let item: FeaturedItem
    @Environment(\.isFocused) var isFocused

    var body: some View {
        ZStack(alignment: .bottomLeading) {
            // Background image — subtle movement
            AsyncImage(url: item.backdropURL) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                Rectangle().fill(.quaternary)
            }
            .scaleEffect(isFocused ? 1.03 : 1.0)

            // Gradient
            LinearGradient(
                colors: [.clear, .clear, .black.opacity(0.8)],
                startPoint: .top,
                endPoint: .bottom
            )

            // Text — moves opposite direction for depth
            VStack(alignment: .leading, spacing: 8) {
                // Logo/title
                Text(item.title)
                    .font(.largeTitle).bold()
                    .foregroundStyle(.white)

                Text(item.tagline)
                    .font(.title3)
                    .foregroundStyle(.white.opacity(0.8))

                HStack(spacing: 20) {
                    Button("Play") { }
                        .buttonStyle(.glassProminent)
                    Button("More Info") { }
                        .buttonStyle(.glass)
                }
                .padding(.top, 8)
            }
            .padding(.leading, 80)
            .padding(.bottom, 60)
            .offset(y: isFocused ? -5 : 0)  // Text lifts toward viewer
        }
        .frame(height: 600)
        .clipShape(RoundedRectangle(cornerRadius: 0))  // Full bleed
        .animation(.spring(response: 0.5, dampingFraction: 0.7), value: isFocused)
        .focusable()
    }
}
```

## 5. Asset creation guidelines

### For layered images (.lsr)

| Specification | Value |
|---------------|-------|
| Layer count | 2-5 layers |
| Max parallax shift | ±15pt foreground, ±5pt background |
| Image overshoot | 20-40pt beyond visible frame per edge |
| Background layer | Should fully cover frame + overshoot |
| Transparency | Required for middle/foreground layers |
| Format per layer | PNG with alpha |
| Final format | .lsr (Image Stack in Asset Catalog) |

### Image sizes

| Usage | Frame size | Background size (with overshoot) |
|-------|-----------|----------------------------------|
| Movie poster (2:3) | 250×375pt | 310×435pt |
| Wide card (16:9) | 350×197pt | 410×257pt |
| Hero banner | 1920×600pt | 1980×660pt |
| App icon | 400×240pt | 460×300pt |

### Design tips

- **Background**: blurred or solid — doesn't need detail
- **Middle**: main subject (character, object) — clear silhouette
- **Foreground**: text, logo, badge — sharp edges
- **Avoid**: thin lines, small text, fine gradients on moving layers
- **Test**: always preview in Parallax Previewer before shipping

## 6. Focus-driven motion

The parallax "tilt" responds to the user's finger position on the Siri Remote touch surface:

| Remote input | Visual response |
|-------------|-----------------|
| Finger at center | Layers at rest position |
| Finger moves left | Layers shift right (background more, foreground less) |
| Finger moves up | Layers shift down |
| Finger lifts off | Layers spring back to rest |
| Focus gained | Scale up + shadow deepens |
| Focus lost | Scale down + shadow lightens |

**Note**: The actual tilt-based parallax (responding to finger position) is only available with **layered images** (.lsr) or `UIImageView.adjustsImageWhenAncestorFocused`. Custom SwiftUI parallax using `isFocused` only responds to focus/unfocus — not continuous finger movement.

For continuous parallax in SwiftUI, you'd need a UIKit bridge:

```swift
// UIKit bridge for continuous parallax
class ParallaxImageView: UIView {
    let imageView = UIImageView()

    override init(frame: CGRect) {
        super.init(frame: frame)
        imageView.adjustsImageWhenAncestorFocused = true
        imageView.clipsToBounds = false
        addSubview(imageView)
    }
}
```

## 7. Rules and anti-patterns

### DO:
- Use layered images (.lsr) when possible — best performance, best effect
- Keep parallax subtle — max 15pt shift for foreground
- Include overshoot on all layers (20-40pt beyond frame)
- Use spring animations (response: 0.3-0.5, damping: 0.6-0.8) for focus transitions
- Test parallax on actual Apple TV hardware
- Scale focused elements 1.05-1.10x
- Deepen shadows on focus (radius 5→20, opacity 0.2→0.5)

### DON'T:
- Don't apply parallax to every element — it's for featured/poster content
- Don't use parallax on text-only elements (no depth layers to shift)
- Don't create more than 5 layers (diminishing returns, performance cost)
- Don't forget background layer overshoot — reveals ugly edges during tilt
- Don't make parallax shift too large (>20pt feels jarring)
- Don't use linear animations for focus — always use spring
- Don't apply parallax to glass elements (glass has its own effects)
- Don't skip the shadow change on focus — it's essential for the depth illusion