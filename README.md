# SingleMoleculeLocalization

Fast estimators for high (and low!) density 2D [single molecule localization microscopy](https://en.wikipedia.org/wiki/Super-resolution_microscopy#Points_accumulation_for_imaging_in_nanoscale_topography_%28PAINT%29) loosely inspired by [ADCG](https://arxiv.org/abs/1507.01562).

!!! warning

    This package uses the squared loss! This means it is only appropriate for relatively high SNR images with *zero-mean* backgrounds,
    i.e. those generated by [TIRF](https://en.wikipedia.org/wiki/Total_internal_reflection_fluorescence_microscope) microscopy.

    You may need to apply a variance-stabilizing transform in addition to removing any background.

## Simulation

A [`ForwardModel`](#) describes the parameters of single molecule localization microscopy (SMLM) experiment.
For square images and an isotropic Gaussian point-spread function a model can be constructed as

{cell=example output=false result=false}
```julia
using SingleMoleculeLocalization
using PlotlyJS
use_style!(:seaborn)
model = ForwardModel(1.5, 16)
```

This models a 16 by 16 image patch with an integrated Gaussian PSF with standard deviation of 1.5 pixels.

!!! warning

    Because `ForwardModel` uses statically-sized arrays, performance degrades *rapidly* with increasing patch size.

A single point source is represented as a [`PointSource`](#), with three
properties. `x` and `y` are the location of the source within an image, while
`intensity` is the brightness.

{cell=example output=false result=false}
```julia
p = PointSource(1.0, 7.5, 4.7)
```
constructs a unit-intensity point source at the spatial location (2.5, 3.7).

We can apply the forward model to a point source to generate a noiseless image:
{cell=example}
```julia
img = model(p)
plot(heatmap(z = img))
```

It's also easy to generate a noisy image of a collection of sources:
{cell=example}
```julia
true_sources = [PointSource(3.23, 2.12, 12.12), PointSource(3.0, 2.5, 2.8), PointSource(4.0, 8.0, 7.0)]
noiseless_img = model(true_sources)
img = noiseless_img + 0.01*randn(16,16)
plot(heatmap(z = img))
```

## Localizing small image patches

For small patches (less than about 20 by 20) we provide an efficient estimator,
`PatchLocalizer`.


{cell=example}
```julia
patchlocalizer = PatchLocalizer(model)
est_sources = patchlocalizer(img, 5, 1E-1)
s_true = scatter(;x=getproperty.(true_sources, :x).-0.5, y=getproperty.(true_sources, :y).-0.5, mode="markers")
s_est = scatter(;x=getproperty.(est_sources, :x).-0.5, y=getproperty.(est_sources, :y).-0.5, mode="markers", marker = attr(symbol = "cross"))
plot([heatmap(z = img), s_true, s_est])
```

The two arguments to `patchlocalizer` are the maximum number of source the algorithm will estimate and the minimum drop in the
loss function the algorithm will accept when adding a new source.
When the drop in the squared loss function from adding a new source falls below `1E-1`,
 the algorithm will return the previously estimated sources.

## Localizing in large images

For larger images we provide the `ImageLocalizer` type:

{cell=example output=false result=false}
```julia
localizer = ImageLocalizer(1.5, 1E-1)
```

`ImageLocalizer`'s can comfortably handle large images:

{cell=example}
```julia
# Generate a random image
img = 0.01*randn(512, 512)
for i in 1:350
    source = PointSource(5.0+randn(), 7+rand(), 7+rand())
    x,y = rand(1:(512-15)),rand(1:(512-15))
    img[x:x+15, y:y+15] .+= model(source)
end
# Localize
est_sources = localizer(img)
s_est = scatter(;x=getproperty.(est_sources, :x).-0.5, y=getproperty.(est_sources, :y).-0.5, mode="markers", marker = attr(size = 3.0, symbol = "cross", color="red"))
plot([heatmap(z = img), s_est])
```
