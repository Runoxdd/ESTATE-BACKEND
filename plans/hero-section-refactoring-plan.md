# Hero Section Refactoring Plan

## Problem Analysis

### Current Issues

1. **Cluttered Two-Column Layout**
   - The hero section uses `grid-template-columns: 1fr 1fr` placing text and image side-by-side
   - This creates visual clutter and doesn't utilize the full width effectively

2. **Excessive Image Wrapper Height**
   - Current: `min-height: 500px` with `aspect-ratio: 4 / 3`
   - Results in a tall, narrow image container that crowds the layout

3. **Image Not Behind Text**
   - Image is in a separate grid column, not layered behind content
   - Missing the modern overlay aesthetic seen in premium real estate sites

### Current Structure

```
hero-container (grid 1fr 1fr)
  hero-content (text, stats, search)
  hero-visual (image wrapper with floating cards)
```

---

## Proposed Solution

### Approach: Background-Image with Overlay

Convert the hero section to a **full-width background image** layout with text content overlaid on top. This creates a modern, immersive hero experience.

### New Structure

```
hero-section (full-width container)
  hero-background (absolute positioned background image)
    image with gradient overlay
  hero-content (centered text overlay)
    badge, title, description, search, stats
  floating-cards (positioned over background)
```

---

## Implementation Details

### 1. HTML Structure Changes

**Before:**
```jsx
<section className="hero-section">
  <div className="hero-container">
    <div className="hero-content">...</div>
    <div className="hero-visual">
      <div className="hero-image-wrapper">
        <img src="/bg.png" alt="Modern Architecture" />
        <div className="image-overlay" />
        <div className="floating-card card-1">...</div>
        <div className="floating-card card-2">...</div>
      </div>
    </div>
  </div>
</section>
```

**After:**
```jsx
<section className="hero-section">
  {/* Background Image Layer */}
  <div className="hero-background">
    <img src="/bg.png" alt="Modern Architecture" className="hero-bg-image" />
    <div className="hero-bg-overlay" />
  </div>

  {/* Content Layer */}
  <div className="hero-container">
    <div className="hero-content">...</div>
  </div>

  {/* Floating Cards Layer */}
  <div className="hero-floating-elements">
    <div className="floating-card card-1">...</div>
    <div className="floating-card card-2">...</div>
  </div>
</section>
```

### 2. SCSS Style Changes

#### Hero Section Container

```scss
.hero-section {
  position: relative;
  min-height: 100vh;
  display: flex;
  align-items: center;
  overflow: hidden;
}
```

#### Background Layer (New)

```scss
.hero-background {
  position: absolute;
  inset: 0;
  z-index: 0;

  .hero-bg-image {
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: center;
  }

  .hero-bg-overlay {
    position: absolute;
    inset: 0;
    // Gradient overlay for text readability
    background: linear-gradient(
      135deg,
      rgba(var(--bg-primary-rgb), 0.85) 0%,
      rgba(var(--bg-primary-rgb), 0.7) 50%,
      rgba(var(--bg-primary-rgb), 0.6) 100%
    );
    
    // Dark theme adjustment
    [data-theme="dark"] & {
      background: linear-gradient(
        135deg,
        rgba(10, 10, 11, 0.9) 0%,
        rgba(10, 10, 11, 0.75) 50%,
        rgba(10, 10, 11, 0.65) 100%
      );
    }
  }
}
```

#### Content Layer

```scss
.hero-container {
  position: relative;
  z-index: 1;
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 $space-8;
  
  @include mobile {
    padding: 0 $space-4;
  }
}

.hero-content {
  max-width: 680px;
  
  // Ensure text is readable over background
  .hero-title,
  .hero-description {
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }
}
```

#### Floating Cards (Repositioned)

```scss
.hero-floating-elements {
  position: absolute;
  inset: 0;
  z-index: 2;
  pointer-events: none;
  
  .floating-card {
    position: absolute;
    pointer-events: auto;
    
    &.card-1 {
      bottom: 20%;
      right: 5%;
    }
    
    &.card-2 {
      top: 20%;
      right: 10%;
    }
  }
}
```

---

## Responsive Behavior

### Desktop (1024px+)
- Full background image visible
- Content left-aligned with max-width
- Floating cards positioned on right side

### Tablet (768px - 1023px)
- Background image covers full width
- Content centered
- Floating cards scaled down

### Mobile (< 768px)
- Background image covers full width
- Content stacked and centered
- Floating cards hidden or repositioned to bottom

```scss
@include mobile {
  .hero-section {
    min-height: auto;
    padding: $space-16 0;
  }
  
  .hero-floating-elements {
    display: none; // Or reposition below content
  }
  
  .hero-content {
    text-align: center;
    max-width: 100%;
  }
}
```

---

## Visual Comparison

### Before
```
+----------------------------------+
|  [TEXT CONTENT]  |  [IMAGE]     |
|                  |              |
|  Title           |  +--------+  |
|  Description     |  |  img   |  |
|  Search          |  |        |  |
|  Stats           |  +--------+  |
|                  |              |
+----------------------------------+
```

### After
```
+----------------------------------+
|  [BACKGROUND IMAGE WITH OVERLAY] |
|                                  |
|   Title                          |
|   Description                    |
|   Search                         |
|   Stats                    [card]|
|                       [card]     |
+----------------------------------+
```

---

## Benefits

1. **Cleaner Layout** - Single focal point with text overlaying image
2. **Better Use of Space** - Image fills container width
3. **Modern Aesthetic** - Matches premium real estate site patterns
4. **Improved Readability** - Gradient overlay ensures text contrast
5. **Responsive Flexibility** - Easier to adapt for different screen sizes

---

## Files to Modify

| File | Changes |
|------|---------|
| `client/src/routes/homePage/homePage.jsx` | Restructure hero section HTML |
| `client/src/routes/homePage/homePage.scss` | Update hero section styles |

---

## Implementation Steps

1. **Refactor JSX Structure**
   - Move image outside of grid layout
   - Create background layer with overlay
   - Reposition floating cards

2. **Update SCSS Styles**
   - Remove two-column grid
   - Add absolute positioning for background
   - Update z-index layering
   - Adjust responsive breakpoints

3. **Test Across Breakpoints**
   - Verify desktop layout
   - Check tablet responsiveness
   - Validate mobile experience

---

## Alternative Approach: CSS Background-Image

If the image doesn't need semantic meaning (SEO), consider using CSS `background-image`:

```scss
.hero-section {
  background-image: url('/bg.png');
  background-size: cover;
  background-position: center;
}
```

This is simpler but less accessible for screen readers and harder to manage with art direction (different images for different breakpoints).