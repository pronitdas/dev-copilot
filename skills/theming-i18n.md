# Theming and Internationalization

## Core Principle

**Design system first, custom UI second.**

## Theme Structure

```
┌─────────────────────────┐
│ Theme                   │
├─────────────────────────┤
│ - Colors                │
│   - Primary             │
│   - Secondary           │
│   - Accent              │
│   - Background          │
│   - Text                │
├─────────────────────────┤
│ - Typography            │
│   - Font family         │
│   - Font sizes          │
│   - Line heights        │
├─────────────────────────┤
│ - Spacing               │
│   - Grid                │
│   - Insets              │
│   - Stack               │
├─────────────────────────┤
│ - Components            │
│   - Buttons             │
│   - Cards               │
│   - Forms               │
└─────────────────────────┘
```

## Component Structure

All UI components follow this pattern:

```tsx
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'text';
  size: 'small' | 'medium' | 'large';
  disabled?: boolean;
  loading?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}

const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false,
  onClick,
  children,
}) => {
  // Implementation using design system tokens
};
```

## MapLibre Theming

Map styles follow a layered approach:

1. Base layer (terrain, water)
2. Reference layer (roads, boundaries)
3. Data layer (user data)
4. Interaction layer (highlights, selection)

```json
{
  "map": {
    "base": {
      "water": "#c9e6ff",
      "land": "#f4f4f0",
      "forest": "#d4e6c6"
    },
    "reference": {
      "road": {
        "major": "#ffffff",
        "minor": "#f8f8f8",
        "stroke": "#e0e0e0"
      }
    },
    "data": {
      "sequential": ["#f7fbff", "#deebf7", "#c6dbef"],
      "categorical": ["#4e79a7", "#f28e2c", "#e15759"]
    }
  }
}
```

## i18n Strategy

- All text uses translation keys
- No hardcoded strings
- RTL support for all components
- Number/date formatting follows locale

## Translation Structure

```json
{
  "en": {
    "common": {
      "buttons": {
        "submit": "Submit",
        "cancel": "Cancel",
        "save": "Save"
      }
    }
  },
  "es": {
    "common": {
      "buttons": {
        "submit": "Enviar",
        "cancel": "Cancelar",
        "save": "Guardar"
      }
    }
  }
}
```
