# PNG Restart Marker specification

Version 0.1

This extension introduces restart markers for [PNG](https://www.w3.org/TR/PNG/) images
with a new chunk and a backward-compatible encoding method,
it enables parallel encoding and decoding of image data.

PNG Restart Marker is backwards-compatible with PNG;
any PNG decoder should be able to display images produced with this encoding method.

## Terminology

"Segment" is a horizontal slice of the image and corresponding subset of image data that can be independently
encoded and decoded.

"Recoverable error" is any error encountered during parallel decoding that should be resumed with conventional decoding.

## Error Handling

Most, if not all errors from parallel decoding should result in falling back to linear decoding,
the simplest way to do this is to continue decoding from the end of the first segment
if any of the worker threads report an error.

The `mARK` chunk is not a critical chunk, if it's invalid it should be ignored by default.

## Structure

A PNG with restart markers is a normal PNG stream with backward-compatible encoding constraints
and an additional chunk type describing encoding metadata.

The IDAT stream is split into segments with each segment corresponding to a subset of the image.
For compression method 0 (DEFLATE) each segment ends with a full flush marker,
allowing each segment of the image data to be decompressed in parallel.

To eliminate data dependencies between segments the first scanline of each segment's
filter type is restricted to algorithm's that do not reference the previous scanline,
allowing each segment to be encoded and decoded in parallel.

### Interoperation with Animated PNG ([APNG](https://wiki.mozilla.org/APNG_Specification))

This extension does not attempt to segment the frames of Animated PNG's
and has no effect on any subsequent frames after the IDAT stream defined by APNG.

## `mARK` Image restart marker

The four-byte chunk type field contains the decimal values:

```
109 65 82 75
```

the `mARK` chunk contains:

```
Segmentation method   1 byte
Segmentation type     1 byte
Segment count         4 bytes
Offset                4 bytes
....                 
```

The segmentation method entry defines the segmentation method used.
The only value defined in this specification is 0 ([Default](#default-segmentation-method)).

The segmentation type entry defines the segmentation type used, for segmentation method 0
this shall be 0 or 1.

Segment count is the number of independent image data segments,
this shall be less than the image height.

Depending on the segmentation type there may be a series of offsets after the segment count entry,
the remaining bytes shall be divisible by 4.
 
If present, the `mARK` chunk shall appear only once before the first IDAT chunk.

### Default segmentation method

Given segment count `N` the image is divided equally into `N` horizontal segments,
if the image height is not evenly divisible the first segment shall be taller by the remainder of the division.

Image data for each segment shall decompress independently of preceding image data.

All segments shall start with the IDAT chunk header and the first scanline of each segment.

For filter method 0 the filter type of the first scanline for each segment shall be 0 (None) or 1 (Sub).
This restriction does not apply to the first segment.

Excluding the last segment all segments shall include the exact amount of image data.

This segmentation method is only defined for interlace method 0.

### Segmentation types

Segmentation method 0 defines two segmentation types as listed in [Table 1](#table-1-segmentation-types).

`Offset(x)` denotes the encoded segment offset defined by [segmentation type 0](#segmentation-type-0).

`Segment(x)` denotes the offset of the segment in the file.

`IDAT(x)` denotes the offset of the IDAT chunk header where `x` is the ordinal number of the chunk in the IDAT stream.

#### Table 1 - Segmentation types

| Type | Offset encoding                           | Offset reconstruction                     |
|------|-------------------------------------------|-------------------------------------------|
| 0    | `Offset(x) = Segment(x + 1) - Segment(x)` | `Segment(x) = Segment(x - 1) + Offset(x)` |
| 1    | N/A                                       | `Segment(x) = IDAT(x)`                    |

#### Segmentation type 0

A PNG decoder determines the number of offset entries from the segment count entry,
the remaining bytes after the segment count entry shall be divisible by 4.

Offsets are relative to the previous segment.
The encoded offsets shall be non-zero and not exceed `2^31-1`.

The first offset corresponds to the second segment of the image and is relative to
the first IDAT chunk.

The first segment starts with the first IDAT chunk and has no corresponding offset.

All offsets shall point to IDAT chunk headers and the first scanline for each segment.


#### Segmentation type 1

No offsets are defined for this type, the number of offsets shall be 0.

Each segment shall be encoded as a single IDAT chunk.

The first IDAT chunk corresponds to the first segment.

A PNG decoder determines the segments offsets by scanning forward in the IDAT stream.

## Recommendations for encoders

* Encoding with restart markers will increase file size,
it should be used in cases where the relative size increase is negligible.
* The number of restart markers affect the overall size increase,
in general one marker (two segments) will double encoding and decoding speed.

## Security Considerations

* Decoders must take care to avoid overflow when calculating offsets;
* The calculated offsets must be verified to be within the bounds of the file.
* When decoding in parallel one thread should not access the segment assigned to another thread.

## Conformance

A PNG encoder or decoder conforms to this specification if it satisfies the following conditions.

### Conformance of PNG encoders

* It does not write a `mARK` chunk or attempt to segment the image if the interlace method is not 0.
* It does not encode relative offsets larger than `2^31-1`.

### Conformance of PNG decoders

* It parses all chunks before the first IDAT before attempting to decode in parallel.
* It recovers from parsing errors when decoding in parallel and can fall back to conventional decoding.
* It treats extra trailing image data in segments other than the last as a recoverable error.
* Encoded offsets in the `mARK` chunk larger than `2^31-1` are treated as invalid.

## References

* https://www.w3.org/TR/PNG/ - Portable Network Graphics (PNG) Specification (Second Edition)
* https://wiki.mozilla.org/APNG_Specification - Animated PNG (APNG) Specification
* https://github.com/libspng/png-restart-marker - Official extension repository
