# QuadMap

Variable resolution maps to better represent spatial uncertainty.

| :warning: WARNING :warning:                                                  |
|:----------------------------------------------------------------------------:|
| This repository has been migrated to https://gitlab.com/spadarian/quadmap-rs |

## Problem

Uncertainty assessment is an integral component of spatial modelling, not only from a analytical point of view, but also as a communication tool. However, end-users find it difficult to perceive uncertainty maps alongside the prediction map. In addition, there is a common misconception that finer resolution maps necessarily have higher precision.

## Solution

Here, we present an approach to take advantage of users' perceptions of the resolution-quality relationship by incorporating prediction and uncertainty into a single, variable resolution digital map where the uncertainty is encoded as the pixel size. We use the [quadtree](https://en.wikipedia.org/wiki/Quadtree) algorithm to recursively partition the original map, aggregating pixels with uncertainty greater than a target threshold. In the resulting maps, users can immediately see where the uncertainty is large as it corresponds to coarser "pixelated" areas.

![QuadMap](https://gitlab.com/spadarian/quadmap-rs/-/raw/master/assets/map.gif)

## Setup

QuadMap is written in [Rust](https://www.rust-lang.org/) and it depends on [GDAL](https://gdal.org/) to read the raster maps, so follow their corresponding instructions to install them in your system.

```console
git clone https://gitlab.com/spadarian/quadmap-rs
cd quadmap-rs
cargo install --path .
quadmap --help
```

## Usage

The input map to process should be a 2-bands file where the first band corresponds to the prediction and the second to the uncertainty expressed as standard deviation. The second requirement is a target standard deviation threshold which will depend on the application.

```console
quadmap map_in.tif -t 0.1 -o map_out.tif --with-stats
```

Since raster maps do not support variable pixel size, this will generate a map using the input map as a template. This means that the output map with have the same dimensions and resolution than the input map but the pixels within one of the "super-pixels" will have the same value. Future versions of QuadMap will include a binary format that will preserve the internal data structure.

## Pixels covariance

In a geospatial context like this, the pixels are not independent to each other and their covariance depends on the distance between them. To describe this distance-dependent relationship, it is possible to use spatial autocorrelation techniques common in geostatistics. QuadMap uses the correlogram to describe how the correlation (Moran's I) between points varies depending on the distance.

To include the correlogram, it is possible to use the `--correlogram` option to pass the curve that describes it.


```console
quadmap map_in.tif -t 0.1 -o map_out.tif --with-stats --correlogram="BSpline(\
    [0., 0., 0., 0., 0., 0., 2884.29743888, 2884.29743888, 2884.29743888, 2884.29743888, 2884.29743888, 2884.29743888],\
    [0.98083363, 0.67429281, -0.03774803, 0.04785177, 0.14373, 0.01608206, 0., 0., 0., 0., 0., 0.],\
    5)"
```

The parameters of the `BSpline` curve are the output of the [`scipy.interpolate.splrep`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.interpolate.splrep.html) function. To use this function, you will need the mean distance and correlation between points at different lags. We provide a python function that can be used to obtain those values from your dataset.

## Usage notes

- The algorithm stops partitioning the super-pixels when their uncertainty is lower than the target. This means that all the super-pixels will have an uncertainty greater than or equal to the target. In practice, you usually want to ensure that a large proportion (e.g. 90%) of the super-pixels have an uncertainty lower than or equal to the target, so you need to __set a lower target value__. The specific value will depend on the map but usually 1/2 of the target uncertainty should be a good start. You can run the process multiple times until you get the value that you want. It is recommended to run it without the output map (omitting the `-o` option) so the process is faster.
