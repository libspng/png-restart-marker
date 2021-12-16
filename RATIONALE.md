# Rationale

## Why two kinds of segmentation types?

One may be easier to implement than the other, depending on which existing PNG library is used.

The final specification may define only one type.

## Why offsets?

It makes it possible to set up each worker thread at the specified offset without having to scan forward
and (depending on implementation) reset to the first IDAT chunk.

Offsets are relative to the previous segment which means each segment can be up to `2^31-1` bytes long.

## Why parse up to the IDAT chunk first?

All ancillary chunks should be parsed before the IDAT stream because it may affect the final image (e.g. the `tRNS` transparency chunk).

This is also the reason why the offset for the first segment is not defined, the offset is known as soon as the first IDAT is reached.

## Why is single-threaded fallback required?

The `mARK` chunk only contains encoding metadata, a corrupt optional chunk should not prevent
an otherwise conformant PNG from being displayed.

It is trivial to encode in a PNG standard conformant way but have it fail to decode in parallel,
all it takes is a one-byte offset of the image data in one segment.

## Segments shall include the exact amount of image data

Extra trailing data in segments other than the last can alter the final image when decoded separately,
if the extra data is ignored single-threaded decoders may get a different final image,
therefore it is not safe to ignore and should be treated as a recoverable error.

The last segment is excluded from the rule because it can be safely ignored.