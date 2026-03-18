# UI Visibility Fixes Plan

## Overview
This plan addresses multiple UI visibility and layout issues affecting the PrimeNest application on PC and mobile views.

---

## Issues Identified

### 1. Sign In / Sign Up Page Not Visible on PC View
**Problem:** The login and register pages are not visible on PC view, being covered by a huge image.

**Root Cause Analysis:**
- The [`auth-page`](client/src/routes/login/login.scss:12) uses a 2-column grid layout
- The [`auth-image-section`](client/src/routes/login/login.scss:312) contains a large background image
- On PC view, the image section may be overlapping or hiding the form section due to:
  - Z-index conflicts between form and image sections
  - The image section not being properly constrained to its grid cell
  - Potential overflow issues with the image wrapper

**Files Affected:**
- [`client/src/routes/login/login.scss`](client/src/routes/login/login.scss)
- [`client/src/routes/register/register.scss`](client/src/routes/register/register.scss)

**Proposed Fix:**
1. Ensure proper z-index layering - form section should have higher z-index than image section
2. Add explicit positioning constraints to prevent image overflow
3. Verify grid cell containment for both sections

---

### 2. Navbar Not Sticky on Mobile and PC
**Problem:** The navbar scrolls away instead of remaining fixed at the top of the viewport.

**Root Cause Analysis:**
- The [`navbar`](client/src/components/navbar/navbar.scss:8) has `position: sticky; top: 0;`
- The [`layout-navbar`](client/src/routes/layout/layout.scss:30) wrapper also has `position: sticky; top: 0;`
- Potential issues:
  - Parent container overflow settings may break sticky behavior
  - Double sticky declaration causing conflicts
  - Missing `will-change` or `transform` properties for GPU acceleration
  - The navbar height changes on scroll (64px → 56px) which may cause reflow issues

**Files Affected:**
- [`client/src/components/navbar/navbar.scss`](client/src/components/navbar/navbar.scss)
- [`client/src/routes/layout/layout.scss`](client/src/routes/layout/layout.scss)

**Proposed Fix:**
1. Remove duplicate sticky positioning from layout-navbar wrapper
2. Ensure no parent containers have `overflow: hidden` or `overflow: auto`
3. Add `will-change: transform` for better performance
4. Consider using `position: fixed` as fallback if sticky continues to fail

---

### 3. Dashboard Logout Button Shifting Outside Container
**Problem:** The logout button on the profile/dashboard page shifts outside its container to the right on PC screen.

**Root Cause Analysis:**
- The [`user-actions`](client/src/routes/profilePage/profilePage.scss:171) container uses flexbox
- Buttons have `flex: 1` distribution
- Potential issues:
  - Button text may be too long causing overflow
  - Missing `overflow: hidden` or text truncation
  - Incorrect padding/margin calculations
  - The sidebar width (320px) may be too narrow for the button content

**Files Affected:**
- [`client/src/routes/profilePage/profilePage.scss`](client/src/routes/profilePage/profilePage.scss)

**Proposed Fix:**
1. Add `min-width: 0` to flex items to allow shrinking
2. Add `overflow: hidden` and `text-overflow: ellipsis` for button text
3. Adjust button padding for better fit
4. Consider reducing font size or using icons only on smaller screens

---

### 4. Light Grey Text Against Light Orange Background
**Problem:** Light grey text is hard to read against light orange/peach backgrounds on the chatbot page and home page.

**Root Cause Analysis:**
- The design uses `var(--text-secondary)` and `var(--text-tertiary)` for various text elements
- The accent color is peach (`#F9B17A`) with light backgrounds
- Specific locations:
  - [`hero-badge`](client/src/routes/homePage/homePage.scss:149) on home page
  - [`feature-card`](client/src/routes/assistantPage/assistantPage.scss:155) descriptions on chatbot page
  - Various text elements with `var(--text-tertiary)` class

**Files Affected:**
- [`client/src/routes/homePage/homePage.scss`](client/src/routes/homePage/homePage.scss)
- [`client/src/routes/assistantPage/assistantPage.scss`](client/src/routes/assistantPage/assistantPage.scss)
- [`client/src/styles/_design-system.scss`](client/src/styles/_design-system.scss) (potentially)

**Proposed Fix:**
1. Change text color to white (`#FFFFFF`) or dark (`#1A1D2E`) on orange/peach backgrounds
2. Increase contrast ratio to meet WCAG AA standards (4.5:1 for normal text)
3. Add specific overrides for text on accent-colored backgrounds
4. Consider using `var(--text-inverse)` for text on accent backgrounds

---

## Implementation Order

1. **Fix 1: Login/Register Page Visibility** (High Priority)
   - Affects user authentication flow
   - Critical for user acquisition

2. **Fix 2: Navbar Sticky Behavior** (High Priority)
   - Affects navigation on all pages
   - Critical for user experience

3. **Fix 3: Dashboard Logout Button** (Medium Priority)
   - Affects user logout functionality
   - Important for user account management

4. **Fix 4: Text Visibility** (Medium Priority)
   - Affects readability and accessibility
   - Important for user experience and compliance

---

## Technical Details

### Fix 1: Login/Register Page

```scss
// In login.scss
.auth-page {
  display: grid;
  grid-template-columns: 1fr 1fr;
  min-height: calc(100vh - 64px);
  background: var(--bg-primary);
  position: relative; // Add positioning context
  
  .auth-form-section {
    position: relative;
    z-index: 2; // Ensure form is above image
    // ... rest of styles
  }
  
  .auth-image-section {
    position: relative;
    z-index: 1; // Image behind form
    overflow: hidden; // Contain the image
    
    .auth-image-wrapper {
      position: absolute;
      inset: 0;
      
      img {
        width: 100%;
        height: 100%;
        object-fit: cover;
      }
    }
  }
}
```

### Fix 2: Navbar Sticky

```scss
// In layout.scss
.layout-navbar {
  position: sticky;
  top: 0;
  z-index: 100;
  flex-shrink: 0;
  // Remove duplicate sticky from navbar component
}

// In navbar.scss
.navbar {
  position: relative; // Changed from sticky
  // Sticky handled by parent wrapper
  will-change: transform; // GPU acceleration
}
```

### Fix 3: Dashboard Logout Button

```scss
// In profilePage.scss
.user-actions {
  display: flex;
  gap: $space-3;
  width: 100%;
  
  .action-btn {
    flex: 1;
    min-width: 0; // Allow shrinking
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    
    span {
      overflow: hidden;
      text-overflow: ellipsis;
    }
  }
}
```

### Fix 4: Text Visibility

```scss
// For elements on orange/peach backgrounds
.hero-badge {
  color: white; // Changed from var(--accent-primary)
  // or use dark text for better contrast
}

.feature-card {
  p {
    color: var(--text-primary); // Darker text for readability
  }
}
```

---

## Testing Checklist

- [ ] Login page visible on PC view (Chrome, Firefox, Safari)
- [ ] Register page visible on PC view (Chrome, Firefox, Safari)
- [ ] Login/Register visible on mobile view
- [ ] Navbar stays fixed on scroll (PC)
- [ ] Navbar stays fixed on scroll (Mobile)
- [ ] Logout button contained within sidebar (PC)
- [ ] Logout button contained within sidebar (Tablet)
- [ ] Text readable on home page hero section
- [ ] Text readable on chatbot/assistant page
- [ ] All changes work in both light and dark themes

---

## Files to Modify

| File | Changes |
|------|---------|
| `client/src/routes/login/login.scss` | Fix z-index and positioning for auth pages |
| `client/src/routes/register/register.scss` | Inherit fixes from login.scss |
| `client/src/components/navbar/navbar.scss` | Fix sticky positioning |
| `client/src/routes/layout/layout.scss` | Ensure proper sticky container |
| `client/src/routes/profilePage/profilePage.scss` | Fix logout button overflow |
| `client/src/routes/homePage/homePage.scss` | Fix text visibility on accent backgrounds |
| `client/src/routes/assistantPage/assistantPage.scss` | Fix text visibility |
