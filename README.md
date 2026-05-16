# Mosaic Reconstruction

Panoramic image stitching in MATLAB. I calibrate a phone camera, detect
Harris corners across overlapping shots, match features, estimate a
projective homography per pair with RANSAC, chain the transforms into a
shared canvas, and alpha-blend the warped images into a single mosaic.

Built for RSN Lab 5 (Robotic Sensing and Navigation, MS Robotics @
Northeastern) and kept here as the cleanest small computer vision project
in my portfolio: the math is short, the failure mode is honest, and the
results speak.

## Results

Three datasets, three different overlap regimes, three different outcomes.

| Dataset            | Frames | Overlap | Outcome                                  |
| :----------------- | :----: | :-----: | :--------------------------------------- |
| Forsyth St. mural  |   8    |   50%   | Clean stitch across the full mural       |
| Ruggles mural      |   5    |   15%   | Stitch holds after raising corner budget |
| Campus brick wall  |   5    |   50%   | Fails: repetitive texture, no unique features |

The brick wall is the interesting one. Harris corners are dense on a
brick face but they all look alike, so matching is ambiguous and the
homography fit collapses. The report (`Mosaic Reconstruction Project
Report.pdf`) shows the four blow-up variants I tried before concluding
the feature is wrong for the scene, not the parameters.

The mural results, the Harris corner overlays, the calibration reprojection
error scatter, and the failure cases are all in the PDF report.

## Pipeline

```
capture overlapping frames
        |
        v
camera calibration  ->  K, distortion (one-time, checkerboard rig)
        |
        v
Harris corner detection (per frame, tiled 3x3, top 9000)
        |
        v
descriptor extraction + matchFeatures (unique pairs)
        |
        v
estimateGeometricTransform2D ('projective', RANSAC, conf 99.9)
        |
        v
chain T(n) = T(n) * T(n-1) * ... * T(1)
        |
        v
re-center on middle image, compute panorama bounds
        |
        v
imwarp each frame + AlphaBlender (binary mask) -> mosaic
```

Notable choices:

- **Harris over SURF / ORB.** The lab brief called for a feature I built
  myself. The Harris implementation here uses the smallest-eigenvalue
  response and tiles the image into a 3x3 grid so corners spread out
  instead of clustering on one high-contrast region.
- **Projective, not affine.** Shots are close to the wall, so parallax
  matters; an affine model wouldn't square up edges at the seams.
- **Re-center on the middle frame.** Composing T(n) onto T(1) by itself
  pushes the canvas off to one side; inverting the middle frame's
  transform and right-multiplying every other transform keeps the
  panorama centered and bounded.
- **AlphaBlender, binary mask.** Simple, fast, and good enough for these
  scenes. No multi-band or graph-cut seam optimization; the report calls
  this out as a next step.

## Camera calibration

Calibration was done once on a 6x9 checkerboard (20mm squares) with 15
poses. Results (`Project Files/Camera Calibration.txt`):

```
Focal length     fc = [1578.96, 1587.69] +/- [21.93, 21.81]
Principal point  cc = [762.26, 1042.91]  +/- [7.35, 13.99]
Skew             alpha_c = 0
Distortion       kc = [0.112, -0.323, 0.003, 0.002, 0]
Reprojection     err = [0.42, 0.46] px
```

Sub-pixel reprojection error, so the intrinsics are trustworthy for the
warp step.

## Repo layout

```
Project Files/
  Lab5_RSN.mlx          stitching pipeline, 50% overlap mural
  Lab5_RSN_15.mlx       stitching pipeline, 15% overlap mural
  Lab5_RSN_NW.mlx       stitching pipeline, brick wall failure case
  harris.m              Harris corner detector (eig response, tiled, optional subpixel)
  convolve2.m           separable 2D convolution helper
  Calibration_Output.mat   intrinsics + distortion from MATLAB Camera Calibrator
  Camera Calibration.txt   human-readable calibration summary
  Calibration Images/   15 checkerboard poses
  Images/               Forsyth St. mural, 50% overlap, 8 frames
  Images15/             Ruggles mural, 15% overlap, 5 frames
  ImagesNW/             Brick wall failure case, 5 frames
  Test/                 per-image transform snapshots from calibration

Mosaic Reconstruction Project Report.pdf   full write-up with figures
```

The three `.mlx` files are identical in logic. They differ only in the
input directory they point at, which is the simplest way to keep each
dataset's run reproducible.

## Quickstart

Requirements: MATLAB R2021a or newer with the Computer Vision Toolbox
and the Image Processing Toolbox.

1. Clone the repo.
2. Open MATLAB and `cd` into `Project Files/`.
3. Open one of the live scripts (e.g. `Lab5_RSN.mlx`).
4. Edit the `buildingDir` path at the top of the script to point at the
   matching local image folder (`Images/`, `Images15/`, or `ImagesNW/`).
5. Run the script.

The Harris corner overlay, matched features, and final mosaic appear
inline in the live script.

## What I'd do differently next time

- **Multi-band blending.** Visible seam ghosts in the 15% overlap mosaic
  could be cleaned up with Burt-Adelson pyramid blending.
- **Bundle adjustment.** Pairwise chaining accumulates drift across 8+
  frames. A global bundle adjustment over all matched correspondences
  would distribute the error and tighten the panorama.
- **Repetitive-texture handling.** For the brick wall, the right answer
  is a richer descriptor (SIFT, ORB) plus a ratio test on the match
  scores; Harris alone has no way to disambiguate near-identical patches.
- **Auto-discover image order.** The pipeline assumes the input frames
  are already in left-to-right order. A small bag-of-features step would
  drop that assumption.

## Tech

MATLAB, Computer Vision Toolbox, Image Processing Toolbox. Camera
intrinsics from MATLAB's Camera Calibrator app. Phone camera (Pixel,
PXL\_\* filenames) for the mural datasets.

## License

MIT. See `LICENSE`.
