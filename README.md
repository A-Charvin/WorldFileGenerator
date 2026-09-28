# Native World File Generator

Georeferences drone photography from flight telemetry alone. It reads EXIF GPS
and XMP flight data, computes the ground position, scale and heading of every
frame, and writes world file sidecars (.jgw, .pgw, .tfw) plus a projection file
(.prj) beside the untouched originals. ArcGIS Pro and QGIS then place each frame
correctly on load. No pixel resampling, no feature matching, no rewritten
rasters.

**Try it in the browser:** [WorldFileGenerator](https://a-charvin.github.io/WorldFileGenerator/)

The web tool is the primary interface. The Python script is kept for offline or
bulk use inside an ArcGIS Pro environment.

## Two ways to use it

| Path | File | Runs in | Best for |
|------|------|---------|----------|
| Web viewer and generator | `index.html` | Any modern browser | Most users. View a flight and generate sidecars in one window, nothing to install. |
| Python script | `nwfg.py` | ArcGIS Pro Python environment | Offline bulk processing, scripted pipelines, machines without a browser workflow. |

Both produce the same sidecars from the same math, so output from either path is
interchangeable.

## Web viewer and generator

A single HTML file. No install, no server, no Python. It opens imagery in place
through the browser and writes nothing to disk. Generated sidecars leave as a
downloaded zip.

### Viewing

- Load a folder by button or by dragging it anywhere onto the window.
  Subfolders are walked recursively.
- Each frame is drawn with the same world file affine used by the Python script,
  over a satellite, street, or blank basemap. Zoom runs to level 23.
- Frames with existing sidecars load georeferenced immediately. Frames without
  them are reported per file so you know what is missing.

### Generating

- Point it at a folder of raw drone images. It parses EXIF GPS, focal length and
  camera identity, plus XMP RelativeAltitude and FlightYawDegree, directly from
  the file bytes.
- It builds a zip containing the .jgw or .pgw and matching .prj for every
  eligible frame. Unzip so the sidecars land beside the rasters.
- The drone model is auto-detected from EXIF Make and Model against a built-in
  table, with manual override and custom sensor constants. Where EXIF reports a
  focal length, that value is preferred over the table default.
- Frames that already have sidecars are skipped by default, and every skip
  reason is listed per file.

### Session-only transforms

Click a frame to select it. A slider rotates it in 0.1 degree steps, and a move
mode lets you drag it. Both affect only the current browser session and are
discarded when the tab closes. Use them to test a correction visually before
committing it in GIS.

### Requirements

Any modern web browser. Folder picking uses the File System Access API in Chrome
and Edge; other browsers fall back to a standard folder input. The map libraries
load from CDNs on first open, so that first load needs a network connection.
Sidecar generation itself runs fully local.

### Access

Open [https://a-charvin.github.io/WorldFileGenerator/](https://a-charvin.github.io/WorldFileGenerator/) in a browser, or download
`index.html` and open the file directly.

## Python script (offline alternative)

Writes the same sidecars in bulk, for environments where a browser workflow does
not fit.

### What it does

For each image the tool:

1. Reads EXIF GPS latitude and longitude from GDAL metadata.
2. Reads XMP flight telemetry (RelativeAltitude, FlightYawDegree) from the
   embedded XMP packet, falling back to EXIF altitude when XMP is absent.
3. Computes ground sample distance from altitude and camera constants.
4. Projects the camera position to UTM with arcpy, or with a built-in
   transverse Mercator implementation when arcpy is unavailable.
5. Builds the six world file coefficients for a rotated, north-referenced
   placement.
6. Writes a world file and a .prj named to match the raster extension.

Drop the sidecars beside the originals and the images load georeferenced. The
pixels are never touched.

### Why sidecars instead of rewritten rasters

- Source JPEGs stay byte identical, so nothing is lost to recompression.
- No resampling means no interpolation blur and no black border wedges from
  rotation.
- Files stay tiny: a world file is six lines of text.
- Placement remains editable. Open the Georeferencing tool in ArcGIS and nudge;
  your edits layer on top of the telemetry placement.

### Requirements

Runs inside the Python environment that ships with ArcGIS Pro. Uses only:

- osgeo (GDAL), bundled with Pro, for raster dimensions and metadata
- arcpy, optional, for coordinate projection (pure Python fallback included)
- Python standard library

No pip installs.

### Usage

1. Copy the script anywhere.
2. Edit `raw_img_dir` and `save_dir` at the bottom of `main()`.
3. Run it with the ArcGIS Pro Python interpreter:

 ```
   "C:\Program Files\ArcGIS\Pro\bin\Python\envs\arcgispro-py3\python.exe" nwfg.py
 ```
4. Keep the sidecars in the same folder as the rasters, same base names.
5. Add the rasters to a map. They land in place.

## Shared math

Ground sample distance in metres per pixel:
```
gsd = (relative_altitude * SENSOR_WIDTH_MM) / (FOCAL_LENGTH_MM * image_width_px)
```
World file affine in row-down convention, with alpha as the azimuth of the image
top edge taken from FlightYawDegree:

```
X = Acol + Brow + C
Y = Dcol + Erow + F
A = gsd * cos(alpha)
B = -gsd * sin(alpha)
D = -gsd * sin(alpha)
E = -gsd * cos(alpha)
C = E0 - (width/2 * A + height/2 * B)
F = N0 - (width/2 * D + height/2 * E)
```


where E0, N0 are the UTM coordinates of the camera position. The determinant
`A*E - B*D` must be negative for every valid world file. Both implementations
assert this so a sign error can never ship a mirrored raster.

Validation: coefficients were checked against a hand-digitized ArcGIS
georeferencing of the same frames. Scale agreed to 0.03 percent and rotation
matched FlightYawDegree directly.

The UTM zone is derived from longitude. The web generator derives the zone and
WKT automatically for any location. The Python script currently ships a fixed
WGS 84 UTM zone 18N WKT with an EGM96 vertical datum; replace `WKT_PRJ` when
flying outside that zone.

## Camera constants

Defaults target the DJI Mavic 4 Pro Hasselblad camera:

- `SENSOR_WIDTH_MM = 17.3` (4/3 CMOS active width)
- `FOCAL_LENGTH_MM = 14.45` (matches EXIF FocalLength, 28 mm equivalent)

In the Python script, edit these two constants at the top of the file. In the web
tool, use the drone model dropdown or the custom fields.

## Output naming

| Raster     | World file | Projection |
|------------|------------|------------|
| .jpg/.jpeg | .jgw       | .prj       |
| .png       | .pgw       | .prj       |
| .tif/.tiff | .tfw       | .prj       |

## Accuracy and limitations

- Placement accuracy is bounded by consumer GNSS, typically one to three metres
  horizontal. Expect to nudge frames in GIS for survey-grade work.
- The transform is affine. Pitch and roll tilt leave a small trapezoidal
  residual, about one to two percent across a frame at typical attitudes, which
  an affine world file cannot express.
- Scale uses relative altitude above takeoff and assumes terrain near takeoff
  elevation. Frames without XMP fall back to EXIF GPS altitude, which is
  ellipsoid height; both tools flag those frames.
- Web generation needs EXIF GPS in the file. Images without EXIF are skipped
  with a per-file reason.

## Provenance

XMP telemetry handling concepts from cheny124800/Drone-Image-Stitching.
Coefficient conventions were reverse engineered from and verified against
ArcGIS Pro 3.6 georeferencing output. Developed for the County of Frontenac.
