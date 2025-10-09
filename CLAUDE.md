# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file, zero-dependency web application for GPS-based cycling navigation. The app loads GPX files and provides real-time GPS tracking with visual feedback including gradient-colored route rendering, compass heading, and dashboard metrics.

**Key characteristic**: This is a single `index.html` file containing all HTML, CSS, and JavaScript. No build process, no dependencies, no package.json.

## How to Run

Simply open `index.html` in a web browser. The app requires:

- HTTPS or localhost (for GPS/compass APIs)
- Location permissions
- Device with GPS capability (for tracking features)

## Architecture

### Coordinate System

The app uses **UTM (Universal Transverse Mercator)** coordinates internally for all map rendering and distance calculations:

- **GPS coordinates** (lat/lon) → converted to UTM via `latLonToUtm(lat, lon)` (lines 905-972)
- **All map rendering** uses UTM easting/northing with inverted Y-axis (`-northing`)
- **Distance calculations** use Cartesian distance in UTM space via `cartesianDistance(p1, p2)`
- GPX points are converted to UTM once on load and stored as `{lat, lon, ele, utm: {easting, northing, zone, hemisphere}}`

This allows precise 2D map rendering and distance calculations without repeated haversine calculations.

### Core State Structure

Single global `state` object (lines 1019-1083) contains:

- **`gpx`**: Loaded GPX track data
  - `points[]`: Array of `{lat, lon, ele, utm}` objects
  - `bounds`: Min/max easting/northing for fit-to-bounds
  - `utmZone`: UTM zone for entire track
  - `totalDistance`: Pre-calculated track distance

- **`tracking`**: Active GPS tracking state
  - `isActive`: Whether GPS tracking is running
  - `totalDistance`: Distance travelled along route
  - `previousSnap`: Last snapped position on track (for delta calculations)
  - `isOffRoute`: >100m from track

- **`position`**: Current user position
  - `current`: Raw GPS position
  - `currentUtm`: GPS position in UTM
  - `snapped`: `{point, index}` - closest point on GPX track
  - `heading`: Smoothed compass heading
  - `speedHistory[]`: For speed smoothing
  - `interpolated`: Predicted position between GPS updates

- **`map`**: Map view state
  - `mode`: "north-up" or "heading-up"
  - `viewBox`: SVG viewport `{x, y, width, height}` in UTM meters
  - `zoomLevel`: Index into `CONSTANTS.ZOOM_SCALES`

- **`interpolation`**: GPS interpolation state
  - `pathSegment[]`: UTM points for predicted path
  - Uses speed history to predict position between GPS updates
  - Enables smooth position updates at high frame rates

### Key Functions

**GPS Tracking Flow** (lines 1636-1921):

1. `updatePosition()` - Called at `settings.updateInterval` (default 2000ms)
2. Gets raw GPS → converts to UTM → finds `snapped` position on track
3. Calculates distance moved, speed, gradient at current segment
4. Starts interpolation if significant movement detected
5. Updates dashboard with metrics including current gradient
6. Calls `updateMap()` to re-render

**GPX Rendering** (`renderGpxTrack()`, lines 1443-1463):

- Iterates through GPX point pairs
- Calculates gradient between each pair: `(elevationChange / horizontalDistance) * 100`
- Colors each segment using `getGradientColor(gradient)` based on `GRADIENT_COLORS` thresholds
- Pre-calculates total track distance

**Map Rendering** (`updateMap()`, lines 2159-2221):

- Updates SVG viewBox based on zoom level and center position
- Positions user marker at current/snapped/interpolated position
- Rotates map for "heading-up" mode
- Scales marker size to maintain constant screen size

**Position Snapping** (`findClosestPointOnGpx()`, lines 1497-1518):

- Finds nearest point on any GPX segment to current GPS position
- Returns `{point, distance, index}` where index is segment number
- Used for both on-route tracking and off-route detection

### Compass & Heading

Multiple heading sources combined:

- **Magnetic heading**: From device compass (DeviceOrientationEvent)
- **GPS heading**: From GPS velocity vector (when moving >12 km/h)
- **Position heading**: Calculated from movement along GPX track
- **Magnetic declination**: Calculated from lat/lon lookup table (lines 1096-1133)
- **Grid convergence**: UTM grid vs true north correction

Final heading uses GPS for calibration, falls back to compass when stationary.

### Gradient Display

Current gradient (added to dashboard):

- Calculated from the GPX segment the user is currently on (`state.position.snapped.index`)
- Uses same color coding as track rendering (`GRADIENT_COLORS`, lines 1132-1193)
- Updated in real-time during GPS tracking
- Shows "-" when not tracking

### Internationalization

`i18n` object (lines 1205-1305) with `ko` and `en` translations. Current language in `state.settings.language`. All UI labels use `data-i18n` attributes.

## Code Patterns

**No frameworks**: Pure vanilla JS with:

- `$()` helper for `querySelector`
- Manual DOM manipulation
- Direct event listeners

**SVG rendering**:

- All map content rendered as SVG
- GPX track is `<path>` elements in `#gpx-track-container`
- User path trail is single `<path>` with accumulated points
- Marker is `<polygon>` triangle

**Settings persistence**: LocalStorage for user preferences (GPS interval, language, etc.)

**Wake lock**: Uses Screen Wake Lock API to prevent screen sleep during tracking

## Development Notes

When modifying this file:

- Test on mobile devices with actual GPS
- Verify in both "north-up" and "heading-up" map modes
- Check gradient color accuracy against elevation profiles
- Ensure UTM zone consistency (track must be in single zone)
- Test with GPX files of varying lengths and elevation profiles
