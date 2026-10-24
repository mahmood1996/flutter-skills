---
name: figma-to-flutter
description: Use when converting Figma designs to Flutter via Figma MCP.
version: 1.0.0
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    category: flutter
    tags: [flutter, figma, design-system, tokens, mcp, widgets]
    related_skills: [feature-spec, feature-build, flutter-clean-architecture]
compatibility: Requires the Figma MCP server (design context, variable definitions, screenshot and asset export tools) and a Flutter project.
---

# Figma to Flutter

Turn Figma designs into Flutter code by first extracting **design tokens** into
`design_system/`, then building **pure widgets** that use only those tokens.
Feature code consumes `design_system/`; `design_system/` never consumes feature code.

## When to use this skill

Use it whenever the user:
- shares a Figma link or node, or mentions Figma together with Flutter;
- asks to implement a screen or component from a design;
- asks to extract design tokens or a design system from Figma;
- wants to sync Flutter tokens with Figma changes.

Use it even if the user never says "design system".

## 1. Folder structure

```
lib/
├── core/
└── design_system/
    ├── assets.dart            # every asset path used in the design
    ├── app_radiis.dart        # every border radius used in the design
    ├── app_colors.dart        # every color (primary, secondary, ..., etc.)
    ├── app_spacing.dart       # every spacing between widgets
    ├── app_paddings.dart      # every padding used inside/around widgets
    ├── app_font_styles.dart   # every font style used in the design
    └── widgets/               # common widgets shared across features
```

If `design_system/` already exists, **extend it**. Never recreate or overwrite
existing tokens; reuse them when a Figma value matches.

## 2. Dependency rules (hard constraints)

These rules are not negotiable. Check them before finishing every task.

1. `design_system/` must **not** depend on any feature. It may depend only on `core/` (and Flutter/Dart SDK, plus UI packages such as `flutter_svg`).
2. `design_system/` contains **pure widgets**. A widget must not read or depend on any state management class (`Bloc`, `Cubit`, `Provider`, `Riverpod`, `GetX`, etc.). No `BlocBuilder`, `context.read<...>()`, and so on.
3. `design_system/` must **not** depend on APIs, databases, use cases, repositories, or models from features. It contains only design tokens and pure widgets.
4. Widgets receive everything through constructor parameters (data and callbacks like `onPressed`). State lives in the feature that uses the widget.
5. Features import from `design_system/`, never the other way around.

Quick check after generating code:

```bash
grep -rnE "features/|flutter_bloc|bloc|cubit|provider|riverpod|dio|http|usecase|repository" lib/design_system/
```

The result must be empty (or only legitimate `core/` imports).

## 3. Workflow

### Step 1: Understand the request
- Get the Figma URL or node ID from the user. If missing, ask for it once.
- Check the existing `design_system/` and `core/` so you know what already exists.

### Step 2: Read the design through the Figma MCP
Use the Figma MCP tools (names vary by server version, so check the tools you have):

1. **Design context / metadata** of the node: layout tree, auto layout settings, sizes, text, and component structure.
2. **Variable definitions**: Figma variables and styles (colors, spacing, radii, typography). Prefer these as the source of truth for token names.
3. **Screenshot** of the node, to visually verify the result later.
4. **Asset export**: icons, images, and illustrations used in the node.

If the MCP returns React/Tailwind/CSS-like output, treat it as a **description of the design**, not as code to port. Translate it to idiomatic Flutter.

### Step 3: Extract tokens into `design_system/`
Collect every unique value used by the node, then write it to the right file (see section 4). Rules:

- **Name by role, not by value**: `primary`, `surface`, `textSecondary` (not `blue500` or `grey333`). If Figma has a variable name, base the token name on it.
- **No duplicates**: if an existing token has the same value, reuse it.
- **No magic numbers in widgets**: every color, radius, spacing, padding, and text style in a widget must come from a token.
- Keep tokens sorted logically (for example spacing from small to large).
- If a value has no clear role, add the token with the closest meaningful name and mention it in the final summary so the user can rename it.

### Step 4: Export assets
- Save exported files under `assets/` (for example `assets/icons/`, `assets/images/`), grouped by type.
- Add each path to `assets.dart`. Never hardcode an asset path in a widget.
- Make sure `pubspec.yaml` lists the asset folders; add them if missing.
- Prefer SVG for icons and vectors, PNG/WebP for raster images.

### Step 5: Build widgets
Decide where each piece of UI goes:

| Piece | Location |
|---|---|
| Used (or likely to be used) in more than one feature: buttons, text fields, cards, app bars, chips, dialogs | `design_system/widgets/` |
| Specific to a single screen/feature | The feature's own `presentation/widgets/` (outside `design_system/`) |
| Whole screens | The feature's own `presentation/` |

Widget guidelines:
- Prefer `StatelessWidget`. Use `StatefulWidget` only for purely visual local state (focus, animation, toggling an obscure-text icon), never for business state.
- Use `const` constructors wherever possible.
- Expose parameters for content, callbacks, and meaningful variants (for example `isLoading`, `isEnabled`), not for every pixel value.
- Map Figma auto layout to Flutter: horizontal -> `Row`, vertical -> `Column`, gap -> spacing tokens (`SizedBox` or the `spacing` parameter), padding -> `AppPaddings`, fill container -> `Expanded`, hug contents -> `MainAxisSize.min`.
- Use theme-independent tokens from `design_system/` directly. Do not hardcode `Color(0xFF...)`, `BorderRadius.circular(12)`, `EdgeInsets.all(16)`, or `TextStyle(...)` in widgets.
- Make it responsive: avoid fixed widths and heights unless the design requires them.
- Add `Semantics` or `semanticLabel` for icons and images that convey meaning.

### Step 6: Verify
1. Compare the result with the Figma screenshot (layout, spacing, colors, typography, radii).
2. Run the dependency check from section 2.
3. Run `dart format .` and `flutter analyze`. Fix any warnings you introduced.
4. Confirm there are no hardcoded colors, radii, spacing, paddings, text styles, or asset paths left in the widgets.

### Step 7: Report
Finish with a short summary:
- Tokens added or reused (per file).
- Assets exported.
- Widgets created and where they live.
- Any assumptions or tokens whose names the user may want to review.

## 4. Token file conventions

Each token file is a class with a private constructor and `static const` members,
so it cannot be instantiated and tokens are compile-time constants.

### `app_colors.dart`
```dart
import 'package:flutter/material.dart';

abstract final class AppColors {
  // Brand
  static const Color primary = Color(0xFF6750A4);
  static const Color secondary = Color(0xFF625B71);

  // Surface
  static const Color background = Color(0xFFFFFBFE);
  static const Color surface = Color(0xFFFFFFFF);

  // Text
  static const Color textPrimary = Color(0xFF1C1B1F);
  static const Color textSecondary = Color(0xFF49454F);

  // Feedback
  static const Color error = Color(0xFFB3261E);
  static const Color success = Color(0xFF2E7D32);
}
```
- Convert Figma colors to `Color(0xAARRGGBB)`; keep opacity in the alpha channel.
- Gradients and shadows go in the same file (`LinearGradient`, `BoxShadow` lists) when the design uses them.

### `app_radiis.dart`
```dart
import 'package:flutter/widgets.dart';

abstract final class AppRadiis {
  static const BorderRadius small = BorderRadius.all(Radius.circular(4));
  static const BorderRadius medium = BorderRadius.all(Radius.circular(8));
  static const BorderRadius large = BorderRadius.all(Radius.circular(16));
  static const BorderRadius full = BorderRadius.all(Radius.circular(999));
}
```

### `app_spacing.dart`
Spacing is the gap **between** widgets.
```dart
import 'package:flutter/widgets.dart';

abstract final class AppSpacing {
  static const double xs = 4;
  static const double sm = 8;
  static const double md = 16;
  static const double lg = 24;
  static const double xl = 32;

  // Ready-made gaps
  static const SizedBox verticalSm = SizedBox(height: sm);
  static const SizedBox horizontalSm = SizedBox(width: sm);
}
```

### `app_paddings.dart`
Paddings are the space **inside or around** a widget.
```dart
import 'package:flutter/widgets.dart';

abstract final class AppPaddings {
  static const EdgeInsets allSm = EdgeInsets.all(8);
  static const EdgeInsets allMd = EdgeInsets.all(16);
  static const EdgeInsets screenHorizontal = EdgeInsets.symmetric(horizontal: 16);
  static const EdgeInsets buttonContent =
      EdgeInsets.symmetric(horizontal: 24, vertical: 12);
}
```

### `app_font_styles.dart`
```dart
import 'package:flutter/widgets.dart';

abstract final class AppFontStyles {
  static const String _fontFamily = 'Inter';

  static const TextStyle headlineLarge = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 32,
    height: 40 / 32, // line height / font size
    fontWeight: FontWeight.w700,
    letterSpacing: 0,
  );

  static const TextStyle bodyMedium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 14,
    height: 20 / 14,
    fontWeight: FontWeight.w400,
  );
}
```
- Convert Figma line height (px) to Flutter `height` as `lineHeight / fontSize`.
- Do **not** bake a color into font styles unless the design always pairs them. Apply color at the usage site with `copyWith(color: AppColors.textPrimary)`.
- If the design uses a custom font, add it to `pubspec.yaml` (or use `google_fonts`) and tell the user.

### `assets.dart`
```dart
abstract final class Assets {
  // Icons
  static const String iconHome = 'assets/icons/home.svg';
  static const String iconCart = 'assets/icons/cart.svg';

  // Images
  static const String imageOnboarding = 'assets/images/onboarding.png';
}
```

## 5. Naming rules

- Files: `snake_case`. Classes: `UpperCamelCase`. Members: `lowerCamelCase`.
- Token classes use the exact names above (`AppColors`, `AppRadiis`, `AppSpacing`, `AppPaddings`, `AppFontStyles`, `Assets`).
- Prefix shared widgets with `App` (`AppButton`, `AppTextField`, `AppCard`) to avoid clashing with Flutter's own widgets.
- Name query methods and getters with **nouns**, not verbs (for example `label`, `backgroundColor`, not `getLabel`, `computeBackgroundColor`).
- Do not end type names with "er" or "or" (for example prefer `AppButtonStyle` over `AppButtonBuilder`, and avoid names like `ColorResolver` or `ThemeProvider`). Choose a noun that describes what the thing *is*.

## 6. Syncing with Figma changes

When the user says the design changed:
1. Re-read the node and variable definitions through the MCP.
2. Diff against the existing tokens: update changed values in place, add new tokens, and flag (do not silently delete) tokens that no longer appear in the design.
3. Update affected widgets and re-run the verification step.

## 7. Common mistakes to avoid

- Hardcoding values in widgets "just this once".
- Naming tokens by value (`red`, `grey16`) instead of by role.
- Putting feature-specific widgets in `design_system/widgets/`.
- Importing a Bloc/Cubit, model, or repository into a design system widget.
- Pasting MCP-generated React/CSS structure into Dart instead of translating it into idiomatic Flutter.
- Overwriting an existing `design_system/` file instead of extending it.
- Exporting the same asset twice under different names.

