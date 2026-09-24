# Mandelbulb

## A beautiful representation of a mandelbulb

[Website](https://mandelbulb-mauve.vercel.app/)

<img width="1280" height="720" alt="mandelbulb" src="https://github.com/user-attachments/assets/59c86e72-a087-4206-981a-442ae359721a" />

I took Sean Bradley's beautiful TSL code and put it in Sveltekit. 

```bash
npx sv create mandelbulb

```

Added the bulb rotation and wobble as well as applied a color gradient.

## Algorithm

The algorithm for a Mandelbulb calculates a three-dimensional analog to the 2D Mandelbrot set by iteratively applying the power formula\(Z = Z^n + C\) using spherical coordinate conversion. 
