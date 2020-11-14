# scan3d-dataset
This repository containts wood scan dataset.

## Camera type
Sick Ruler E12xx

## cameras config
* start fo capturing by `ENABLE` signal
* sync by rotary encoder

## directory name format

`<tree><class>c_xz_<step>[_<num>]` where:
* `tree` - wood type
* `class` - diameter class
* `xz` - type of layout
* `step` - rotary encoder step (10 steps ~ 1 mm)
* `num` - № off scan session, optional

## file names
each scan containts `9` files

scan files have prefix `scan_<num>_<side>` where `num` - № of scan in session, `side` - position of camera

## file extentions
* `da2` - containts status map `line` - `status`(`status == 0` means "no errors")
* `dat` - data
* `xml` - information about layout(more info in Sick iCon API help)

## `xz` layout

```cpp
struct Buffer
{
  float X[NUM_LINES][POINTS_PER_LINE];
  float Y[NUM_LINES][POINTS_PER_LINE];
  uint32_t Mark[NUM_LINES][MARKS_PER_LINE];
  uint8_t Intensity[NUM_LINES][POINTS_PER_LINE];
};
```
