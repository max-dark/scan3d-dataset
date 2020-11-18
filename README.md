# scan3d-dataset
This repository containts wood scan dataset.

## Camera type
Sick Ruler E12xx

## cameras config
* start of capturing by `ENABLE` signal
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
## how to load file
c++17
```cpp
// buffer.hxx
#ifndef POINT3D__BUFFER_HXX
#define POINT3D__BUFFER_HXX

#include <vector>
#include <string>

#include <optional>

namespace Scan
{

using std::size_t;

struct Mark
{
    enum Mask
    {
        BOOL_MASK = 1u,
        BYTE_MASK = 0xFFu,
        ENCODER_ENABLE = BOOL_MASK << 30u,
        ENCODER_A = BOOL_MASK << 28u,
        ENCODER_B = BOOL_MASK << 27u,
        OVER_TRIG_SHIFT = 8u,
        OVER_TRIG_MASK = BYTE_MASK << OVER_TRIG_SHIFT,
    };
    enum Index
    {
        VALUE,
        STATUS,
        SAMPLE,
        ENCODER,
        SCAN_ID,
    };

    uint32_t value() const;
    uint32_t sample() const;
    uint32_t encoder() const;
    uint32_t scanId() const;

    [[nodiscard]] bool encoder_enable() const;
    [[nodiscard]] bool encoder_a_phase() const;
    [[nodiscard]] bool encoder_b_phase() const;
    [[nodiscard]] uint8_t over_trig() const;

    static Mark from_data(const uint32_t* data);

private:
    uint32_t m_value;
    uint32_t m_status;
    uint32_t m_sample_timestamp;
    uint32_t m_encoder_timestamp;
    uint32_t m_scan_id;
};

struct Buffer
{
    std::vector<char> data;

    [[nodiscard]] const float * getX(size_t line) const;
    [[nodiscard]] const float * getY(size_t line) const;
    [[nodiscard]] const uint32_t * getMark(size_t line) const;
    [[nodiscard]] const uint8_t * getIntensity(size_t line) const;

    [[nodiscard]] size_t pointsPerLine() const;
    [[nodiscard]] size_t numLines() const;
    [[nodiscard]] bool enable(size_t line) const;

    void swap(Buffer& other);
};

using Optional = std::optional<Buffer>;

Optional loadBuffer(const std::string& file);

} // namespace Scan

#endif //POINT3D__BUFFER_HXX

```
```cpp
// buffer.cxx
#include "buffer.hxx"

#include <filesystem>
#include <fstream>
#include <ios>

namespace fs = std::filesystem;

namespace Scan
{

constexpr auto points_per_line = 1024u;
constexpr auto num_lines = 512u;

constexpr auto points_count = 1024u * 512u;
constexpr auto marks_per_line = 5u;

constexpr auto f_size = points_count * sizeof(float);
constexpr auto m_size = marks_per_line * sizeof(uint32_t);
constexpr auto x_offset = 0u;
constexpr auto y_offset = x_offset + f_size;
constexpr auto m_offset = y_offset + f_size;
constexpr auto i_offset = m_offset + m_size;

static_assert(x_offset == 0u);
static_assert(y_offset == (points_count * sizeof(float)));


[[nodiscard]] const void* getDataAt(const Buffer* buffer, size_t offset)
{
    return buffer->data.data() + offset;
}


template <typename Type>
const Type* getPointer(const Buffer * buffer, size_t offset, size_t line, size_t perLine)
{
    auto raw_ptr = getDataAt(buffer, offset);
    auto ptr = reinterpret_cast<const Type*>(raw_ptr);
    return ptr + line * perLine;
}

template <typename Type>
const Type* getPointer(const Buffer * buffer, size_t offset, size_t line)
{
    return getPointer<Type>(buffer, offset, line, buffer->pointsPerLine());
}


const float *Buffer::getX(size_t line) const
{
    return getPointer<float>(this, x_offset, line);
}

const float *Buffer::getY(size_t line) const
{
    return getPointer<float>(this, y_offset, line);
}

const uint32_t *Buffer::getMark(size_t line) const
{
    return getPointer<uint32_t>(this, m_offset, line, marks_per_line);
}

const uint8_t *Buffer::getIntensity(size_t line) const
{
    return getPointer<uint8_t>(this, i_offset, line);
}

size_t Buffer::pointsPerLine() const
{
    return points_per_line;
}

size_t Buffer::numLines() const
{
    return num_lines;
}

void Buffer::swap(Buffer & other)
{
    std::swap(data, other.data);
}

bool Buffer::enable(size_t line) const
{
    auto m = getMark(line);
    auto mark = Mark::from_data(m);

    return mark.encoder_enable();
}

Optional loadBuffer(const std::string &file)
{
    if (const fs::path file_path{file}; fs::exists(file_path))
    {
        Buffer temp;
        auto file_size = fs::file_size(file_path);
        temp.data.resize(file_size);
        {
            std::ifstream data;
            data.open(file_path, std::ios::binary | std::ios::in);
            if (data.is_open())
            {
                 if (data.read(temp.data.data(), file_size))
                 {
                     return temp;
                 }
            }
        }
    }
    return {};
}

bool Mark::encoder_enable() const
{
    return 0 != (m_status & ENCODER_ENABLE);
}

bool Mark::encoder_a_phase() const
{
    return 0 != (m_status & ENCODER_A);
}

bool Mark::encoder_b_phase() const
{
    return 0 != (m_status & ENCODER_B);
}

uint8_t Mark::over_trig() const
{
    return (m_status & OVER_TRIG_MASK) >> OVER_TRIG_SHIFT;
}

Mark Mark::from_data(const uint32_t *data)
{
    Mark m{};
    m.m_value = data[VALUE];
    m.m_status = data[STATUS];
    m.m_sample_timestamp = data[SAMPLE];
    m.m_encoder_timestamp = data[ENCODER];
    m.m_scan_id = data[SCAN_ID];
    return m;
}

uint32_t Mark::value() const
{
    return m_value;
}

uint32_t Mark::sample() const
{
    return m_sample_timestamp;
}

uint32_t Mark::encoder() const
{
    return m_encoder_timestamp;
}

uint32_t Mark::scanId() const
{
    return m_scan_id;
}
} // namespace Scan

```

python3 + numpy
```python
# num of scan lines per file
NUM_SCANS = 512
# num of points per line
DOTS_PER_SCAN = 1024
# total points per file
TOTAL_DOTS = NUM_SCANS * DOTS_PER_SCAN

SCAN_SHAPE = (NUM_SCANS, DOTS_PER_SCAN)
MASK_SHAPE = (NUM_SCANS, 5)

### define types

# point coords [X, R] относительно камеры(чем меньше R тем дальше от камеры)
# Camera position in coord system (Xcam, Ycam) = (1000.0, 2054.0)
coord_t  = np.dtype((np.float32, SCAN_SHAPE))
# line status
mask_t   = np.dtype((np.int32, MASK_SHAPE))
# intensity
intens_t = np.dtype((np.uint8, SCAN_SHAPE))

# struct data_t
# {
#   float32 X[NUM_SCANS][DOTS_PER_SCAN];
#   float32 Y[NUM_SCANS][DOTS_PER_SCAN];
#   int32_t mask[NUM_SCANS][5]; // line scan status, encoder position and sensors state
#   uint8_t intens[NUM_SCANS][DOTS_PER_SCAN];
# };
data_t = np.dtype([
    ('X', coord_t),
    ('Y', coord_t),
    ('mask', mask_t),
    ('Intensity', intens_t)
])

def load_scan(f_name):
    data = np.fromfile(f_name, dtype=data_t)
    scan = data[0]
    return scan
```
