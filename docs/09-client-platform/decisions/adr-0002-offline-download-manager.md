# ADR-0002: Offline Download Manager Architecture

## Status
Accepted

## Context

Subscribers in markets with unreliable or expensive mobile data connections expressed a strong preference for the ability to download content for offline viewing. This feature is particularly high-value in markets where the majority of streaming occurs on mobile devices over cellular networks.

Implementing offline downloads for DRM-protected content is architecturally complex:

- The DRM license must be obtained at download time and stored securely on the device
- Licenses have a validity period; expired licenses must be renewed when the device reconnects
- Content removed from the Netflix catalog must be handled gracefully (offline content becomes unplayable when the title is removed)
- Device storage must be managed to prevent the download manager from consuming all available storage
- Content must be encrypted at rest using device-specific keys that cannot be transferred to another device

A naive implementation that downloads video files without addressing these constraints creates both a poor user experience and potential security vulnerabilities.

## Decision

Build the Offline Download Manager as a first-class client subsystem with the following design:

**License pre-fetching:** When a download is initiated, the DRM license is fetched immediately and stored in the device's secure storage (Keystore on Android, Secure Enclave on iOS). The license is bound to the specific device and cannot be transferred.

**Content encryption:** Downloaded content is encrypted using device-specific keys derived from the DRM license. The encrypted content is stored in an app-private storage directory that cannot be accessed by other applications.

**License renewal:** The Download Manager monitors the expiry time of stored licenses. When a device comes online and a license is within 7 days of expiry, the manager attempts a silent renewal without user interaction.

**Storage management:** The Download Manager enforces a per-device storage quota (user-configurable within platform limits). When storage is full, the manager prompts the user to delete older downloads before new ones can begin. Automatically deleting content without user consent is not permitted.

**Catalog removal handling:** Titles removed from the Netflix catalog have their licenses invalidated server-side. The next time the Download Manager checks license validity (at app launch or connectivity restoration), the affected content is marked unplayable and the user is notified.

## Consequences

### Positive
- Enables high-quality offline playback without compromising content security
- License pre-fetching means offline playback does not require any network connectivity after the download is complete
- Device-bound encryption prevents content from being copied between devices or extracted from the device's storage
- Storage management and license renewal are handled transparently, reducing friction for subscribers

### Negative
- DRM platform fragmentation (Widevine on Android, FairPlay on iOS, PlayReady on some TV platforms) requires per-platform DRM integration work
- License renewal failures when devices remain offline for extended periods result in content becoming unplayable; this is a known friction point for long trips or travel
- Storage management is a frequent source of user complaints; the interface for managing download storage must be carefully designed

## Alternatives Considered

**Download without DRM pre-fetch (streaming from local storage):** Simpler to implement but requires a network connection to validate the DRM license on each playback attempt. Defeats the purpose of offline for areas with no connectivity.

**Managed device storage (automatic deletion):** Reduces storage management complexity but creates user trust issues; subscribers do not expect content they downloaded to disappear automatically.
