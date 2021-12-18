# Rationale

## Why the name?

It is inspired by the JPEG feature of the same name,
also used to enable parallel processing.
The name is meant to be self-explanatory to advanced users who
are already familiar with the term "restart marker".

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

## Why are interlaced images not supported?

Interlaced images are typically used where bandwidth is limited
and parallel decoding is not possible, due to its very limited
use cases it was deemed not worth the added complexity. 

It is not practical to support interlaced images, deinterlacing would require
locks to prevent concurrent reads and writes to the same row.

Ignoring some corner cases with extremely small images
it is possible to split an interlaced image into two segments,
because the last pass is always the bottom half the image,
which is technically not interlaced.

If an interlaced image were to be split into two segments
the default segmentation method would calculate the size of the bottom half
to the same dimensions and allow parallel decoding with deinterlacing,
this is the only known feasible use case.

## Why a segment count entry?

Although it is possible for the number of `Offset` entries to be derived from the
length of the chunk as is done for some of the standard PNG chunks
it would be awkward for segmentation type 1 to use dummy `Offset`'s to signal the same.

If segmentation type 1 were to be dropped from a future version then the segment count
entry would also be removed.

## Why restrict the filter types for the first scanline?

To guarantee predictable performance and memory usage,
the restrictions also limit complexity, reducing the attack surface of the decoder.

A decoder with optimal memory usage holds two scanlines in memory for defiltering,
usually the processed image rows go into an application-managed buffer.
Aside from a few, small and fixed-size buffers no additional memory is allocated.

If segments were allowed to reference the previous segment's last scanline then
a simple parallel decoder would have to wait for the previous segment to finish,
defeating the purpose of parallel decoding.
Not only would decompression have to finish on the previous segment but defiltering too,
defiltering would require the previous scanline to be defiltered first.

Although it is possible to allocate more memory to buffer the uncompressed segment
it would mean additional complexity for most implementations which is undesirable for decoders and
worker threads would still have to wait for defiltering to finish on the previous segment.
