# CLAUDE.md - Mobile Application Section

This file provides guidance for working with the Mobile Application documentation section.

## Section Purpose

This section documents the ThingsBoard Mobile Application:

- **Customization**: UI customization, mobile actions, OAuth2
- **Development**: Flutter setup, build, release, Firebase integration

## File Structure

```
17-mobile-app/
├── README.md                      # Mobile app overview and architecture
├── mobile-app-customization.md    # UI customization, mobile actions
└── mobile-app-development.md      # Setup, build, release, Firebase
```

## Writing Guidelines

### Audience

Mobile developers building custom ThingsBoard mobile apps. Assume familiarity with mobile development but not necessarily with Flutter or ThingsBoard-specific patterns.

### Content Pattern

Mobile app documents should include:

1. **Overview** - What the feature/component does
2. **Configuration** - Mobile Center settings
3. **Code Examples** - Working Flutter/Dart examples
4. **Build Commands** - Development and release builds
5. **Pitfalls** - Common development issues
6. **See Also** - Related documentation

### Mobile Documentation Pattern

For mobile features:

```markdown
## Feature Name

**Platform**: iOS / Android / Both

### Configuration

| Setting | Description | Location |
|---------|-------------|----------|
| ... | ... | ... |

### Implementation

\`\`\`dart
// Dart example
\`\`\`

### Build Commands

\`\`\`bash
flutter build ...
\`\`\`

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "Mobile Center" for ThingsBoard's app configuration portal
- Use "Bundle" for app package configuration
- Use "Mobile Action" for device capability integration
- Use "configs.json" for app configuration file

### Diagrams

Use Mermaid diagrams to show:

- App architecture (`graph TB`)
- Mobile Center configuration (`graph TB`)
- Push notification flow (`sequenceDiagram`)
- Build workflow (`graph LR`)

### Technology-Agnostic Rule

Focus on configuration and usage, not Flutter internals:

**DO**: "Mobile actions integrate device capabilities like camera and GPS with dashboards"
**DON'T**: "MobileActionHandler extends MethodCallHandler with platform channel invocation"

**DO**: "Push notifications are delivered via Firebase Cloud Messaging"
**DON'T**: "FirebaseMessagingService onMessageReceived() calls NotificationManager.notify()"

**DO**: "The app fetches configuration from Mobile Center on startup"
**DON'T**: "ConfigService calls DioClient.get() to fetch configs.json via REST"

## Reference Sources

When updating this section, cross-reference:

- ThingsBoard Mobile Application GitHub repository
- Flutter official documentation
- Firebase Cloud Messaging documentation

## Related Sections

- `06-api-layer/` - REST API used by mobile app
- `09-security/` - Authentication methods
- `10-frontend/` - Dashboard system

## Common Tasks

### Documenting Mobile Center Configuration

1. Show bundle settings
2. Document OAuth2 configuration
3. Explain navigation layout
4. Include QR code settings

### Documenting Build Process

1. Document environment setup
2. Show development commands
3. Explain release builds
4. Include store submission steps

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/17-mobile-app/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **flutter-expert** | `/flutter-expert` | Flutter development, widget architecture |
| **dart-expert** | `/dart-expert` | Dart language, null safety, async patterns |
| **technical-writer** | `/technical-writer` | Clear mobile documentation |

### When to Use Each Skill

- **Documenting Flutter widgets**: Use `/flutter-expert` for Flutter patterns
- **Documenting Dart code**: Use `/dart-expert` for language features
- **Writing setup guides**: Use `/technical-writer` for clarity

## Key Mobile Concepts

When documenting mobile app, emphasize:

| Concept | Key Points |
|---------|------------|
| **Mobile Center** | Centralized app configuration portal |
| **Bundle Configuration** | App identity, OAuth, navigation |
| **Mobile Actions** | Camera, GPS, QR scanner integration |
| **Push Notifications** | Firebase Cloud Messaging delivery |
| **Dashboard WebView** | Native dashboard rendering |
| **configs.json** | App configuration file |

## Mobile Actions

| Action | Description | Returns |
|--------|-------------|---------|
| Take Photo | Camera capture | Base64 image |
| Take Picture from Gallery | Image picker | Base64 image |
| Scan QR Code | QR scanner | QR code value |
| Get Phone Location | GPS coordinates | Lat/Long |
| Make Phone Call | Phone dialer | - |
| Open Map Location | Map display | - |
| Take Screenshot | Screen capture | Base64 image |

## Version Compatibility

| ThingsBoard | Mobile App | Flutter |
|-------------|------------|---------|
| 4.0.0+ | 1.7.0 | 3.29.0 |
| 3.9.0 | 1.5.0 | 3.24.4 |
| 3.8.0+ | 1.3.0 | 3.22.2 |

## Common Pitfalls to Document

Ensure documentation covers these mobile issues:

| Pitfall | Description |
|---------|-------------|
| Version mismatch | App version not matching platform version |
| Firebase misconfiguration | Push notifications not working |
| OAuth redirect URI | Incorrect callback configuration |
| WebView communication | Dashboard not loading properly |
| Build signing | Missing keystore or provisioning profile |
| Flutter version | Using incompatible Flutter SDK |

## Build Commands

```bash
# Development
flutter run --dart-define-from-file configs.json

# iOS Release
flutter build ipa --no-tree-shake-icons --dart-define-from-file configs.json

# Android Release
flutter build appbundle --no-tree-shake-icons --dart-define-from-file configs.json
```

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
