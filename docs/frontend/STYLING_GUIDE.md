# Frontend Styling Guide

## Overview

PicklesApp uses Tailwind CSS as its primary styling framework, providing a utility-first approach that enables rapid development while maintaining consistency. The styling system includes age-adaptive classes, a comprehensive color system, responsive breakpoints, and component-specific patterns.

## Table of Contents
1. [Tailwind CSS Setup](#tailwind-css-setup)
2. [Age-Adaptive Utility Classes](#age-adaptive-utility-classes)
3. [Responsive Breakpoints](#responsive-breakpoints)
4. [Color System](#color-system)
5. [Typography Scale](#typography-scale)
6. [Spacing System](#spacing-system)
7. [Component Styling Patterns](#component-styling-patterns)
8. [Dark Mode Considerations](#dark-mode-considerations)

---

## Tailwind CSS Setup

### Installation

```bash
# Install Tailwind CSS and dependencies
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Tailwind Configuration

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        // Primary brand colors
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
          950: '#172554',
        },
        
        // Success (money/rewards)
        success: {
          50: '#f0fdf4',
          100: '#dcfce7',
          200: '#bbf7d0',
          300: '#86efac',
          400: '#4ade80',
          500: '#22c55e',
          600: '#16a34a',
          700: '#15803d',
          800: '#166534',
          900: '#14532d',
        },
        
        // Warning (pending/overdue)
        warning: {
          50: '#fffbeb',
          100: '#fef3c7',
          200: '#fde68a',
          300: '#fcd34d',
          400: '#fbbf24',
          500: '#f59e0b',
          600: '#d97706',
          700: '#b45309',
          800: '#92400e',
          900: '#78350f',
        },
        
        // Danger (errors/rejections)
        danger: {
          50: '#fef2f2',
          100: '#fee2e2',
          200: '#fecaca',
          300: '#fca5a5',
          400: '#f87171',
          500: '#ef4444',
          600: '#dc2626',
          700: '#b91c1c',
          800: '#991b1b',
          900: '#7f1d1d',
        },
        
        // Family-specific colors (for multi-household)
        family: {
          blue: '#3b82f6',
          purple: '#a855f7',
          pink: '#ec4899',
          orange: '#f97316',
          green: '#22c55e',
          teal: '#14b8a6',
        },
      },
      
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        display: ['Poppins', 'system-ui', 'sans-serif'],
      },
      
      fontSize: {
        // Age-adaptive sizes (use CSS variables)
        'adaptive-xs': 'var(--font-size-xs)',
        'adaptive-sm': 'var(--font-size-sm)',
        'adaptive-base': 'var(--font-size-base)',
        'adaptive-lg': 'var(--font-size-lg)',
        'adaptive-xl': 'var(--font-size-xl)',
        'adaptive-2xl': 'var(--font-size-2xl)',
        'adaptive-3xl': 'var(--font-size-3xl)',
      },
      
      spacing: {
        'adaptive': 'var(--spacing-base)',
        'adaptive-sm': 'calc(var(--spacing-base) * 0.5)',
        'adaptive-lg': 'calc(var(--spacing-base) * 1.5)',
        'adaptive-xl': 'calc(var(--spacing-base) * 2)',
      },
      
      borderRadius: {
        'adaptive': 'var(--border-radius)',
        'adaptive-sm': 'calc(var(--border-radius) * 0.5)',
        'adaptive-lg': 'calc(var(--border-radius) * 1.5)',
      },
      
      boxShadow: {
        'card': '0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1)',
        'card-hover': '0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1)',
        'button': '0 1px 2px 0 rgb(0 0 0 / 0.05)',
      },
      
      animation: {
        'fade-in': 'fadeIn 0.3s ease-in-out',
        'slide-up': 'slideUp 0.3s ease-out',
        'slide-down': 'slideDown 0.3s ease-out',
        'bounce-subtle': 'bounceSubtle 0.5s ease-in-out',
      },
      
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        slideDown: {
          '0%': { transform: 'translateY(-10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        bounceSubtle: {
          '0%, 100%': { transform: 'scale(1)' },
          '50%': { transform: 'scale(1.05)' },
        },
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}
```

### Global Styles

```css
/* src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Import fonts */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Poppins:wght@600;700;800&display=swap');

@layer base {
  /* Root variables */
  :root {
    /* Colors */
    --color-background: #ffffff;
    --color-foreground: #1f2937;
    --color-muted: #6b7280;
    
    /* Default (tween) sizing */
    --font-size-xs: 12px;
    --font-size-sm: 14px;
    --font-size-base: 16px;
    --font-size-lg: 20px;
    --font-size-xl: 24px;
    --font-size-2xl: 32px;
    --font-size-3xl: 40px;
    
    --spacing-base: 16px;
    --border-radius: 12px;
    --button-height: 48px;
    --touch-target-min: 48px;
  }
  
  /* Base styles */
  html {
    @apply antialiased;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
  }
  
  body {
    @apply bg-gray-50 text-gray-900 font-sans;
  }
  
  /* Smooth scrolling */
  html {
    scroll-behavior: smooth;
  }
  
  /* Focus visible for accessibility */
  *:focus-visible {
    @apply outline-2 outline-offset-2 outline-primary-500;
  }
}

@layer components {
  /* Safe area insets for mobile */
  .safe-area-top {
    padding-top: env(safe-area-inset-top);
  }
  
  .safe-area-bottom {
    padding-bottom: env(safe-area-inset-bottom);
  }
  
  .safe-area-left {
    padding-left: env(safe-area-inset-left);
  }
  
  .safe-area-right {
    padding-right: env(safe-area-inset-right);
  }
}

@layer utilities {
  /* Text truncation */
  .line-clamp-1 {
    display: -webkit-box;
    -webkit-line-clamp: 1;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
  
  .line-clamp-2 {
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
  
  .line-clamp-3 {
    display: -webkit-box;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
  
  /* Scrollbar styles */
  .scrollbar-thin::-webkit-scrollbar {
    width: 8px;
    height: 8px;
  }
  
  .scrollbar-thin::-webkit-scrollbar-track {
    @apply bg-gray-100;
  }
  
  .scrollbar-thin::-webkit-scrollbar-thumb {
    @apply bg-gray-300 rounded;
  }
  
  .scrollbar-thin::-webkit-scrollbar-thumb:hover {
    @apply bg-gray-400;
  }
}
```

---

## Age-Adaptive Utility Classes

### Age Bracket CSS Variables

```css
/* src/styles/age-adaptive.css */

/* Young (5-9 years): Large, playful */
.ui-young {
  /* Typography */
  --font-size-xs: 14px;
  --font-size-sm: 16px;
  --font-size-base: 18px;
  --font-size-lg: 24px;
  --font-size-xl: 32px;
  --font-size-2xl: 40px;
  --font-size-3xl: 48px;
  
  --font-weight-normal: 500;
  --font-weight-medium: 600;
  --font-weight-bold: 700;
  
  --line-height-tight: 1.3;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;
  
  /* Spacing */
  --spacing-xs: 8px;
  --spacing-sm: 12px;
  --spacing-base: 24px;
  --spacing-lg: 36px;
  --spacing-xl: 48px;
  
  /* Components */
  --button-height: 60px;
  --button-min-width: 120px;
  --touch-target-min: 60px;
  --border-radius: 24px;
  --card-padding: 32px;
  
  /* Shadows */
  --shadow-card: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
}

/* Tween (10-13 years): Balanced */
.ui-tween {
  /* Typography */
  --font-size-xs: 12px;
  --font-size-sm: 14px;
  --font-size-base: 16px;
  --font-size-lg: 20px;
  --font-size-xl: 24px;
  --font-size-2xl: 32px;
  --font-size-3xl: 40px;
  
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-bold: 700;
  
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.625;
  
  /* Spacing */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-base: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;
  
  /* Components */
  --button-height: 48px;
  --button-min-width: 100px;
  --touch-target-min: 48px;
  --border-radius: 12px;
  --card-padding: 24px;
  
  /* Shadows */
  --shadow-card: 0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1);
}

/* Teen (14-17 years): Compact, professional */
.ui-teen {
  /* Typography */
  --font-size-xs: 11px;
  --font-size-sm: 13px;
  --font-size-base: 14px;
  --font-size-lg: 18px;
  --font-size-xl: 20px;
  --font-size-2xl: 24px;
  --font-size-3xl: 30px;
  
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-bold: 600;
  
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.625;
  
  /* Spacing */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-base: 12px;
  --spacing-lg: 20px;
  --spacing-xl: 28px;
  
  /* Components */
  --button-height: 40px;
  --button-min-width: 80px;
  --touch-target-min: 44px;
  --border-radius: 8px;
  --card-padding: 16px;
  
  /* Shadows */
  --shadow-card: 0 1px 2px 0 rgb(0 0 0 / 0.05);
}

/* Adult (parents): Same as teen */
.ui-adult {
  @apply ui-teen;
}
```

### Usage in Components

```jsx
// Adaptive wrapper automatically applies class
export function AdaptivePage({ children }) {
  const { ageBracket } = useAgeBracket();
  
  return (
    <div className={`ui-${ageBracket} min-h-screen`}>
      {children}
    </div>
  );
}

// Use adaptive utilities
<button className="
  min-h-[var(--button-height)]
  px-adaptive
  rounded-adaptive
  text-adaptive-base
  font-medium
">
  Click Me
</button>
```

---

## Responsive Breakpoints

### Default Breakpoints

```javascript
// Tailwind default breakpoints
const breakpoints = {
  sm: '640px',   // Mobile landscape, small tablets
  md: '768px',   // Tablets
  lg: '1024px',  // Desktop
  xl: '1280px',  // Large desktop
  '2xl': '1536px', // Extra large desktop
};
```

### Mobile-First Approach

```jsx
// Stack on mobile, grid on tablet+
<div className="
  flex flex-col gap-4
  md:grid md:grid-cols-2 md:gap-6
  lg:grid-cols-3
">
  {items.map(item => <ItemCard key={item.id} {...item} />)}
</div>

// Hide on mobile, show on desktop
<div className="hidden lg:block">
  <Sidebar />
</div>

// Show on mobile, hide on desktop
<div className="block lg:hidden">
  <MobileNav />
</div>
```

### Container Sizes

```jsx
// Responsive container widths
<div className="
  container mx-auto
  px-4 sm:px-6 lg:px-8
  max-w-7xl
">
  {/* Content */}
</div>

// Full width on mobile, constrained on desktop
<div className="w-full lg:max-w-md lg:mx-auto">
  <LoginForm />
</div>
```

---

## Color System

### Primary Brand Colors

```jsx
// Primary blues (brand color)
<button className="bg-primary-600 hover:bg-primary-700 text-white">
  Submit
</button>

// Lighter shade for backgrounds
<div className="bg-primary-50 border border-primary-200">
  Info banner
</div>
```

### Semantic Colors

```jsx
// Success (positive actions, money earned)
<Badge className="bg-success-100 text-success-800">
  Approved
</Badge>

<div className="bg-success-600 text-white">
  +$5.00 earned
</div>

// Warning (pending, overdue)
<Badge className="bg-warning-100 text-warning-800">
  Pending Approval
</Badge>

<div className="bg-warning-50 border-l-4 border-warning-400">
  Chore overdue by 2 days
</div>

// Danger (errors, rejections)
<Badge className="bg-danger-100 text-danger-800">
  Rejected
</Badge>

<div className="bg-danger-50 text-danger-700">
  Error: Failed to save
</div>
```

### Family-Specific Colors

For multi-household support, assign unique colors to each family:

```jsx
// components/layout/FamilyBadge.jsx
const FAMILY_COLORS = {
  blue: 'bg-blue-100 text-blue-800 border-blue-300',
  purple: 'bg-purple-100 text-purple-800 border-purple-300',
  pink: 'bg-pink-100 text-pink-800 border-pink-300',
  orange: 'bg-orange-100 text-orange-800 border-orange-300',
  green: 'bg-green-100 text-green-800 border-green-300',
  teal: 'bg-teal-100 text-teal-800 border-teal-300',
};

export function FamilyBadge({ family }) {
  const colorClass = FAMILY_COLORS[family.color] || FAMILY_COLORS.blue;
  
  return (
    <span className={`
      inline-flex items-center
      px-2.5 py-0.5
      rounded-full text-xs font-medium
      border
      ${colorClass}
    `}>
      {family.name}
    </span>
  );
}
```

### Gradient Backgrounds

```jsx
// Hero gradient
<div className="bg-gradient-to-r from-primary-500 to-primary-700">
  Hero content
</div>

// Card with subtle gradient
<div className="bg-gradient-to-br from-white to-gray-50">
  Card content
</div>

// Success gradient
<div className="bg-gradient-to-r from-success-400 to-success-600">
  Reward earned!
</div>
```

---

## Typography Scale

### Font Families

```jsx
// Sans-serif (body text)
<p className="font-sans">
  Regular body text uses Inter font
</p>

// Display font (headings)
<h1 className="font-display font-bold">
  Bold headings use Poppins
</h1>
```

### Font Sizes

```jsx
// Fixed sizes
<h1 className="text-4xl font-bold">Main Heading</h1>
<h2 className="text-3xl font-semibold">Sub Heading</h2>
<h3 className="text-2xl font-semibold">Section</h3>
<p className="text-base">Body text</p>
<span className="text-sm text-gray-600">Helper text</span>
<span className="text-xs text-gray-500">Caption</span>

// Age-adaptive sizes
<h1 className="text-adaptive-3xl font-bold">
  Scales with age bracket
</h1>
<p className="text-adaptive-base">
  Body text that adapts
</p>
```

### Font Weights

```jsx
<p className="font-light">Light (300)</p>
<p className="font-normal">Normal (400)</p>
<p className="font-medium">Medium (500)</p>
<p className="font-semibold">Semibold (600)</p>
<p className="font-bold">Bold (700)</p>
<p className="font-extrabold">Extra Bold (800)</p>
```

### Text Colors

```jsx
// Primary text
<p className="text-gray-900">Primary text</p>

// Secondary text
<p className="text-gray-600">Secondary text</p>

// Muted text
<p className="text-gray-500">Muted text</p>

// Branded text
<p className="text-primary-600">Link or branded text</p>

// Error text
<p className="text-danger-600">Error message</p>
```

---

## Spacing System

### Padding & Margin

```jsx
// Standard spacing scale
<div className="p-0">No padding</div>
<div className="p-1">4px padding</div>
<div className="p-2">8px padding</div>
<div className="p-3">12px padding</div>
<div className="p-4">16px padding</div>
<div className="p-6">24px padding</div>
<div className="p-8">32px padding</div>

// Adaptive spacing
<div className="p-adaptive">Scales with age bracket</div>
<div className="px-adaptive py-adaptive-sm">Mixed adaptive spacing</div>

// Directional spacing
<div className="pt-4 pr-6 pb-4 pl-6">Individual sides</div>
<div className="px-6 py-4">Horizontal & vertical</div>
```

### Gap (for Flexbox/Grid)

```jsx
// Flex gap
<div className="flex gap-2">
  <button>Button 1</button>
  <button>Button 2</button>
</div>

// Grid gap
<div className="grid grid-cols-3 gap-4">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>

// Adaptive gap
<div className="flex gap-adaptive">
  Spacing scales with age
</div>
```

### Space Between

```jsx
// Space between children
<div className="flex flex-col space-y-4">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>

<div className="flex space-x-2">
  <button>Button 1</button>
  <button>Button 2</button>
</div>
```

---

## Component Styling Patterns

### Buttons

```jsx
// Primary button
<button className="
  inline-flex items-center justify-center
  px-4 py-2
  bg-primary-600 hover:bg-primary-700
  text-white font-medium
  rounded-lg
  transition-colors
  focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2
  disabled:opacity-50 disabled:cursor-not-allowed
">
  Submit
</button>

// Secondary button
<button className="
  inline-flex items-center justify-center
  px-4 py-2
  bg-gray-200 hover:bg-gray-300
  text-gray-800 font-medium
  rounded-lg
  transition-colors
">
  Cancel
</button>

// Danger button
<button className="
  inline-flex items-center justify-center
  px-4 py-2
  bg-danger-600 hover:bg-danger-700
  text-white font-medium
  rounded-lg
  transition-colors
">
  Delete
</button>

// Ghost button
<button className="
  inline-flex items-center justify-center
  px-4 py-2
  bg-transparent hover:bg-gray-100
  text-primary-600 font-medium
  rounded-lg
  transition-colors
">
  Learn More
</button>
```

### Cards

```jsx
// Standard card
<div className="
  bg-white
  rounded-lg
  shadow-card hover:shadow-card-hover
  p-6
  transition-shadow
">
  Card content
</div>

// Card with border
<div className="
  bg-white
  rounded-lg
  border border-gray-200
  p-6
  hover:border-primary-300
  transition-colors
">
  Card content
</div>

// Adaptive card
<div className="
  bg-white
  rounded-adaptive
  shadow-[var(--shadow-card)]
  p-[var(--card-padding)]
">
  Spacing adapts to age bracket
</div>
```

### Inputs

```jsx
// Text input
<input
  type="text"
  className="
    block w-full
    px-3 py-2
    border border-gray-300 rounded-lg
    text-gray-900 placeholder-gray-400
    focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-transparent
    disabled:bg-gray-100 disabled:cursor-not-allowed
  "
  placeholder="Enter text"
/>

// Input with error
<input
  type="text"
  className="
    block w-full
    px-3 py-2
    border-2 border-danger-500 rounded-lg
    text-gray-900
    focus:outline-none focus:ring-2 focus:ring-danger-500
  "
/>

// Select
<select className="
  block w-full
  px-3 py-2
  border border-gray-300 rounded-lg
  bg-white
  text-gray-900
  focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-transparent
">
  <option>Option 1</option>
  <option>Option 2</option>
</select>
```

### Badges

```jsx
// Status badges
<span className="
  inline-flex items-center
  px-2.5 py-0.5
  rounded-full
  text-xs font-medium
  bg-success-100 text-success-800
">
  Approved
</span>

<span className="
  inline-flex items-center
  px-2.5 py-0.5
  rounded-full
  text-xs font-medium
  bg-warning-100 text-warning-800
">
  Pending
</span>

<span className="
  inline-flex items-center
  px-2.5 py-0.5
  rounded-full
  text-xs font-medium
  bg-danger-100 text-danger-800
">
  Rejected
</span>

// Pill badge
<span className="
  inline-flex items-center
  px-3 py-1
  rounded-full
  text-sm font-medium
  bg-primary-100 text-primary-800
  border border-primary-200
">
  New Feature
</span>
```

### Modals

```jsx
// Modal overlay
<div className="
  fixed inset-0
  bg-black bg-opacity-50
  flex items-center justify-center
  p-4
  z-50
">
  {/* Modal content */}
  <div className="
    bg-white
    rounded-lg
    shadow-xl
    max-w-md w-full
    p-6
    animate-slide-up
  ">
    <h2 className="text-xl font-bold mb-4">Modal Title</h2>
    <p className="text-gray-600 mb-6">Modal content</p>
    
    <div className="flex justify-end gap-3">
      <button className="px-4 py-2 bg-gray-200 rounded-lg">
        Cancel
      </button>
      <button className="px-4 py-2 bg-primary-600 text-white rounded-lg">
        Confirm
      </button>
    </div>
  </div>
</div>
```

### Lists

```jsx
// Simple list
<ul className="divide-y divide-gray-200">
  <li className="py-4">Item 1</li>
  <li className="py-4">Item 2</li>
  <li className="py-4">Item 3</li>
</ul>

// Card list
<div className="space-y-4">
  <div className="bg-white rounded-lg shadow p-4">Item 1</div>
  <div className="bg-white rounded-lg shadow p-4">Item 2</div>
  <div className="bg-white rounded-lg shadow p-4">Item 3</div>
</div>

// Grid list
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  <div className="bg-white rounded-lg shadow p-4">Item 1</div>
  <div className="bg-white rounded-lg shadow p-4">Item 2</div>
  <div className="bg-white rounded-lg shadow p-4">Item 3</div>
</div>
```

---

## Dark Mode Considerations

**Note:** Dark mode is a future enhancement, not in initial release.

### Dark Mode Setup (Future)

```javascript
// tailwind.config.js
module.exports = {
  darkMode: 'class', // or 'media'
  // ... rest of config
}
```

### Dark Mode Classes (Future)

```jsx
// Automatic dark mode support
<div className="
  bg-white dark:bg-gray-800
  text-gray-900 dark:text-white
  border-gray-200 dark:border-gray-700
">
  Content that adapts to dark mode
</div>

// Dark mode button
<button className="
  bg-primary-600 dark:bg-primary-500
  hover:bg-primary-700 dark:hover:bg-primary-600
  text-white
">
  Works in both modes
</button>
```

---

## Accessibility

### Focus States

```jsx
// Visible focus ring
<button className="
  focus:outline-none
  focus:ring-2 focus:ring-primary-500 focus:ring-offset-2
">
  Accessible button
</button>

// Focus within (for containers)
<div className="
  border border-gray-300
  focus-within:ring-2 focus-within:ring-primary-500
">
  <input className="focus:outline-none" />
</div>
```

### Screen Reader Only

```jsx
// Hide visually but keep for screen readers
<span className="sr-only">
  This text is only for screen readers
</span>

// Skip to content link
<a href="#main-content" className="
  sr-only
  focus:not-sr-only focus:absolute focus:top-4 focus:left-4
  focus:z-50 focus:px-4 focus:py-2 focus:bg-primary-600 focus:text-white
">
  Skip to content
</a>
```

### ARIA Labels

```jsx
// Button with icon only
<button aria-label="Close modal">
  <XMarkIcon className="w-5 h-5" />
</button>

// Input with error
<div>
  <input
    type="email"
    aria-invalid="true"
    aria-describedby="email-error"
  />
  <p id="email-error" className="text-danger-600 text-sm mt-1">
    Please enter a valid email
  </p>
</div>
```

---

## Animation & Transitions

### Hover Effects

```jsx
// Scale on hover
<button className="
  transition-transform
  hover:scale-105
  active:scale-95
">
  Interactive button
</button>

// Color transition
<a href="#" className="
  text-primary-600
  transition-colors
  hover:text-primary-700
">
  Link
</a>
```

### Loading States

```jsx
// Spinning loader
<div className="animate-spin h-5 w-5 border-2 border-primary-600 border-t-transparent rounded-full" />

// Pulsing placeholder
<div className="animate-pulse bg-gray-200 h-10 rounded" />

// Skeleton loading
<div className="space-y-3">
  <div className="h-4 bg-gray-200 rounded animate-pulse" />
  <div className="h-4 bg-gray-200 rounded animate-pulse w-5/6" />
  <div className="h-4 bg-gray-200 rounded animate-pulse w-4/6" />
</div>
```

### Entry Animations

```jsx
// Fade in
<div className="animate-fade-in">
  Content fades in
</div>

// Slide up
<div className="animate-slide-up">
  Content slides up
</div>

// Staggered list
<div className="space-y-2">
  {items.map((item, i) => (
    <div
      key={item.id}
      className="animate-fade-in"
      style={{ animationDelay: `${i * 100}ms` }}
    >
      {item.name}
    </div>
  ))}
</div>
```

---

## Best Practices

1. **Use utility-first approach** - Compose styles with Tailwind utilities
2. **Extract repeated patterns** - Create components for common patterns
3. **Maintain consistency** - Use design tokens (colors, spacing, etc.)
4. **Optimize for mobile** - Mobile-first responsive design
5. **Consider accessibility** - Proper focus states, ARIA labels, color contrast
6. **Age-adaptive by default** - Use adaptive utilities where appropriate
7. **Keep animations subtle** - Don't distract from content
8. **Test across devices** - Real devices, not just browser dev tools
9. **Document custom patterns** - Help team understand design decisions
10. **Monitor bundle size** - Purge unused styles in production

---

## Performance Tips

### PurgeCSS Configuration

```javascript
// tailwind.config.js
module.exports = {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  // Tailwind will automatically purge unused styles in production
}
```

### Avoid @apply in Production

```css
/* Avoid */
.my-button {
  @apply px-4 py-2 bg-primary-600 text-white rounded;
}

/* Better: Use utilities directly in JSX */
<button className="px-4 py-2 bg-primary-600 text-white rounded">
```

### JIT Mode (Enabled by Default)

Tailwind JIT generates styles on-demand, reducing build size and time.

---

*Last Updated: January 29, 2026*
