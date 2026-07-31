# Design System — Mandatory Rules

Source of truth: [design-system/styles/tokens.css](../../design-system/styles/tokens.css) and [design-system/styles/components.css](../../design-system/styles/components.css)  
Interactive reference: [design-system/DesignSystem.html](../../design-system/DesignSystem.html)  
Screen screenshots: [design-system/screenshots/](../../design-system/screenshots/)

## The One Rule

**Never hardcode a color, size, spacing, or radius value.**  
Always use the Dart token constants: `AppColors`, `AppTypography`, `AppSpacing`, `AppRadius`.  
If a value is not covered by a constant, check `tokens.css` / `components.css` first, then add the constant — never inline the value.

---

## Colors — `AppColors.*`

Brand is **teal `#14B8A6`**, not sage green.

| Token | Hex | `AppColors` constant |
|---|---|---|
| brand-500 | `#14B8A6` | `AppColors.primary` |
| brand-600 | `#0D9488` | `AppColors.primaryDark` |
| brand-700 | `#0F766E` | `AppColors.primaryDeep` |
| brand-50 | `#F0FDFA` | `AppColors.primaryTint` |
| paper | `#FAFAF9` | `AppColors.background` |
| white | `#FFFFFF` | `AppColors.surface` |
| ink-50 | `#F5F5F5` | `AppColors.surfaceSunken` |
| ink-75 | `#EEEEEE` | `AppColors.surfaceHover` |
| ink-900 | `#1A1A1A` | `AppColors.textPrimary` |
| ink-500 | `#666666` | `AppColors.textSecondary` |
| ink-400 | `#8C8C8C` | `AppColors.textTertiary` |
| ink-300 | `#A6A6A6` | `AppColors.textMuted` |
| ink-100 | `#E5E5E5` | `AppColors.border` |
| ink-75 | `#EEEEEE` | `AppColors.borderSubtle` |
| success-500 | `#10B981` | `AppColors.success` |
| success-50 | `#ECFDF5` | `AppColors.successBg` |
| success-700 | `#047857` | `AppColors.successDark` |
| warning-500 | `#F59E0B` | `AppColors.warning` |
| warning-50 | `#FFFBEB` | `AppColors.warningBg` |
| warning-700 | `#B45309` | `AppColors.warningDark` |
| danger-500 | `#EF4444` | `AppColors.danger` |
| danger-50 | `#FEF2F2` | `AppColors.dangerBg` |
| danger-700 | `#B91C1C` | `AppColors.dangerDark` |

### Category Colors

| Category | Foreground | Background |
|---|---|---|
| food | `#E8835C` | `#FBEEE6` |
| transport | `#5B8DC9` | `#E7EEF7` |
| shopping | `#B47ACA` | `#F1E8F5` |
| bills | `#C29438` | `#F6EED9` |
| fun | `#E0628F` | `#FAE6EC` |
| health | `#4FA88B` | `#E0F0E9` |
| fees | `#666666` | `#EEEEEE` |
| other | `#8C8C8C` | `#F5F5F5` |

Use `AppColors.cat<Name>` and `AppColors.cat<Name>Bg` constants.

---

## Typography — `AppTypography.*`

Fonts: **DM Sans** (primary), **DM Mono** (numbers), **Fraunces** (hero display only).

| Style | Font | Size | Weight | `AppTypography` constant |
|---|---|---|---|---|
| Hero amount | DM Mono | 64px | 500 | `AppTypography.amountHero` |
| Flash card amount | DM Mono | 40px | 500 | `AppTypography.amountDisplay` |
| h1 | DM Sans | 28px | 700 | `AppTypography.h1` |
| h2 | DM Sans | 22px | 600 | `AppTypography.h2` |
| h3 | DM Sans | 18px | 600 | `AppTypography.h3` |
| body-lg | DM Sans | 17px | 400 | `AppTypography.bodyLg` |
| body | DM Sans | 15px | 400 | `AppTypography.body` |
| body-sm | DM Sans | 13px | 400 | `AppTypography.bodySm` |
| label (md) | DM Sans | 15px | 500 | `AppTypography.labelMd` |
| label (sm) | DM Sans | 13px | 500 | `AppTypography.labelSm` |
| caption | DM Sans | 12px | 400 | `AppTypography.caption` |
| micro | DM Sans | 11px | 500 | `AppTypography.micro` |

Rules:
- Use `DM Mono` for **all numeric amounts** — never DM Sans for currency values
- Use `Fraunces` only for hero display/marketing contexts, not in functional UI
- Adjust color with `.copyWith(color: AppColors.*)` — never override font or size

---

## Spacing — `AppSpacing.*`

4px base grid. **Never use arbitrary pixel values.**

| Constant | Value |
|---|---|
| `AppSpacing.xs` | 4px |
| `AppSpacing.sm` | 8px |
| `AppSpacing.md` | 12px |
| `AppSpacing.lg` | 16px |
| `AppSpacing.xl` | 20px |
| `AppSpacing.x2l` | 24px |
| `AppSpacing.x3l` | 32px |
| `AppSpacing.x4l` | 40px |
| `AppSpacing.x5l` | 48px |
| `AppSpacing.x6l` | 64px |
| `AppSpacing.x7l` | 80px |
| `AppSpacing.screenPadding` | 20px — use for all screen horizontal padding |

---

## Border Radius — `AppRadius.*`

| Constant | Value | Use case |
|---|---|---|
| `AppRadius.xs` / `AppRadius.xsBR` | 4px | Tight tags, micro elements |
| `AppRadius.sm` / `AppRadius.smBR` | 8px | Small surfaces, soft inputs |
| `AppRadius.md` / `AppRadius.mdBR` | 12px | Inputs (48px height), buttons (default) |
| `AppRadius.lg` / `AppRadius.lgBR` | 16px | Cards, buttons (lg) |
| `AppRadius.xl` / `AppRadius.xlBR` | 20px | Bottom sheet top corners |
| `AppRadius.x2l` / `AppRadius.x2lBR` | 28px | Flash card |
| `AppRadius.full` / `AppRadius.fullBR` | 999px | Chips, badges, FAB, pill shapes |

---

## Shadows — Flutter `BoxShadow`

```dart
// shadow-card (use on cards)
BoxShadow(color: Color(0x0A14181A), blurRadius: 2, offset: Offset(0, 1)),
BoxShadow(color: Color(0x0A14181A), blurRadius: 16, offset: Offset(0, 4)),

// shadow-md (use on elevated sheets/dialogs)
BoxShadow(color: Color(0x0F14181A), blurRadius: 12, offset: Offset(0, 4)),
BoxShadow(color: Color(0x0A14181A), blurRadius: 3, offset: Offset(0, 1)),

// shadow-stack-2 (use on flash card)
BoxShadow(color: Color(0x1414181A), blurRadius: 20, offset: Offset(0, 8)),
BoxShadow(color: Color(0x0A14181A), blurRadius: 6, offset: Offset(0, 2)),
```

---

## Component Specs

### Button — use `AppButton`

| Variant | Height | Radius | Font |
|---|---|---|---|
| default (md) | 44px | `AppRadius.md` (12px) | body, 600 |
| lg | 56px | `AppRadius.lg` (16px) | body-lg, 600 |
| sm | 36px | `AppRadius.sm` (8px) | body-sm, 600 |

- **Primary**: `AppColors.primaryDark` bg, white text
- **Secondary**: white bg, `AppColors.border` border, `AppColors.textPrimary` text
- **Ghost**: transparent bg, `AppColors.textPrimary` text
- **Danger**: transparent bg, `AppColors.dangerDark` text, `AppColors.danger` border
- Disabled: opacity 0.45
- Always use `AppButton` — never a raw `ElevatedButton` or `TextButton`

### Input — use `AppTextField`

- Height: **48px**
- Radius: `AppRadius.md` (12px)
- Border color: `AppColors.border`
- Focus border: `AppColors.primaryDark`, width 1.5px
- Placeholder color: `AppColors.textMuted`
- Always use `AppTextField` — never a raw `TextField` in feature screens

### Chip — use `AppChip`

| State | Height | Bg | Border | Text color |
|---|---|---|---|---|
| Default | 36px | `AppColors.surface` | `AppColors.border` | `AppColors.textPrimary` |
| Selected / last | 36px | `AppColors.primaryTint` | transparent | `AppColors.primaryDeep` |
| Placeholder | 36px | `AppColors.surface` | dashed `AppColors.border` | `AppColors.textTertiary` |
| Filter | 32px | `AppColors.surface` | `AppColors.border` | `AppColors.textPrimary` |
| Filter active | 32px | `AppColors.textPrimary` | — | `AppColors.surface` |

- Radius: always `AppRadius.full`
- Always use `AppChip` — never a private `_Chip` widget

### Card — use `AppCard`

- Radius: `AppRadius.lg` (16px)
- Shadow: shadow-card
- Border: `AppColors.borderSubtle`
- Padding: `AppSpacing.xl` (20px)
- Variants: `elevated` (default), `flat`, `tinted`, `ink`

### Bottom Sheet

- Background: `AppColors.surface`
- Top radius: `AppRadius.xl` (20px) — `BorderRadius.vertical(top: Radius.circular(AppRadius.xl))`
- Drag handle: 36×4px, `AppColors.border`, `AppRadius.full`

### Flash Card (Review)

- Width: 320px, min-height: 520px
- Radius: `AppRadius.x2l` (28px)
- Shadow: shadow-stack-2
- Amount font: `AppTypography.amountDisplay` (DM Mono 40px)

### Keypad

- Key height: 56px
- Radius: `AppRadius.md` (12px)
- Font: DM Mono, 22px, weight 500
- `000` key: smaller font 18px, `AppColors.primaryDeep` text, `AppColors.primaryTint` bg
- Backspace: icon only, `AppColors.textSecondary`
- Press state: scale 0.97, `AppColors.surfaceSunken` bg

### FAB (Capture button in bottom nav)

- Size: 52×52px
- Radius: `AppRadius.full`
- Background: `AppColors.primaryDark`
- Icon: 26×26px, white
- Shadow: `0 8px 22px rgba(15, 118, 110, 0.32)`

### Bottom Nav

- Height: 80px (includes 18px bottom safe padding)
- Background: `AppColors.surface`
- Top border: `AppColors.borderSubtle`
- Active item: `AppColors.primary`
- Inactive item: `AppColors.textTertiary`
- Label: `AppTypography.micro`

---

## Motion / Animation

Use `flutter_animate`. Match these durations:

| Constant | Duration | Use case |
|---|---|---|
| `--dur-instant` | 80ms | Amount display update, key press feedback |
| `--dur-fast` | 160ms | Button state, chip selection, color transitions |
| `--dur-base` | 240ms | Screen transitions, modal appear |
| `--dur-slow` | 400ms | Fade out, success badge disappear |
| `--dur-flip` | 480ms | Flash card flip |

Easing:
- Standard transitions: `Curves.easeInOut`
- Emphasis (enter): `Curves.easeOut` (approx `cubic-bezier(0.2, 0, 0, 1)`)
- Soft: `Curves.easeInOutSine`

---

## Checklist — Before Submitting Any UI Work

- [ ] Zero hardcoded hex colors — all via `AppColors.*`
- [ ] Zero hardcoded font sizes or weights — all via `AppTypography.*`
- [ ] Zero arbitrary spacing numbers — all via `AppSpacing.*`
- [ ] Zero arbitrary radius numbers — all via `AppRadius.*`
- [ ] Used `AppButton`, `AppTextField`, `AppChip`, `AppCard` — not raw Flutter widgets
- [ ] Numeric/currency values use DM Mono (`AppTypography.amount*`)
- [ ] Screen horizontal padding is `AppSpacing.screenPadding` (20px)
- [ ] Bottom sheets use `AppColors.surface` bg + `AppRadius.xl` top corners + drag handle
- [ ] Shadows match one of the defined shadow tokens above
- [ ] Animations use durations from the motion table above
