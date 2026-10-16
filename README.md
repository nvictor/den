# Demo Effect Notation (DEN)

**Version:** 1.0.0

DEN describes how a demo effect generates, transforms, colors, and evolves an image. Chains describe mechanisms, not executable programs, buffer storage, or exact frame timing.

## Reading a chain

```text
cloud: prev(cloud) → blur(5) → perturb(-1,1) → wrap(0,16)
image: cloud → floor → palette
```

- `name:` names the final result. It can be a field, coordinates, geometry, or an image. Names are local to each effect. Define each name once within it; built-in names are reserved.
- `→` passes the result as the next operation's first argument: `field → clamp(0)` means `clamp(field,0)`. It shows data flow, not time.
- Reference a defined name or an explicitly supplied input, such as a palette or region. Forward references are allowed; current-step cycles are not.
- Use `prev` for feedback and explain initial values. Keep evolving state separate from rendered output.
- Omitted arguments remain unspecified, except documented defaults. Explain effect-specific parameters, boundaries, and mappings beside the chain.
- Stages must accept the preceding result's type. For example, `screen → blur` is invalid because blur needs scalar values.

Define animation counters explicitly, such as `t: next(0)`.

`x,y,z` are coordinates and `i` is an element index. Expressions use numbers, ordinary arithmetic, comparisons, and function calls such as `sin`, `cos`, `floor`, and `random`. `field(x,y)` reads a value; `uv` conventionally names sampling coordinates. Field names in expressions use the value at the current location; `sample` reads a field at supplied coordinates.

### Syntax

```text
name: value → operation(arguments) → ...
```

Chains may have no arrows. Operations may omit parentheses when no arguments are written.

Arguments are comma-separated names, expressions, quoted text, tuples, or nested calls. Names start with a letter and contain letters, digits, or underscores.

## Operations

Signatures show all inputs; chains supply the first. Formulas apply at each location. Placeholder names describe arguments, not additional built-ins.

### Coordinates and sources

```text
screen: (all,all)
top: (all,miny)
bottom: (all,maxy)
left: (minx,all)
right: (maxx,all)
all: every coordinate from min to max, inclusive
```

Each `all` spans its axis. Bounds belong to the field's domain; x increases rightward and y downward. These are built-in definitions.

```text
seed(region,value): value at each location in region; undefined elsewhere
```

The caller may supply a named region. Evaluate the value separately at each selected location.

| Source | Meaning |
| :--- | :--- |
| `points(position,count)` | Create count points, evaluating position separately for each point. Position is a scalar for 1D, a pair for 2D, or a triple for 3D. |
| `text(content,position)` | Glyphs from content, evaluating position separately for each glyph. i is the glyph index, starting at 1, and stays with the glyph through later transformations. |

### Transformations

```text
linear((x,y)): a*x + b*y + c
sample((x,y),field): field(x,y)
radial(position,center,factor): center + (position - center) * factor
```

- `linear`: explain horizontal rate a, vertical rate b, and starting value c beside the effect.
- `sample`: explain fractional coordinates and bounds beside the effect.
- `radial`: position and center are coordinate pairs. Factor 1 leaves positions unchanged; below 1 moves toward the center; above 1 moves away.

```text
rotate(position,center,angle):
  position = (x,y)
  center = (cx,cy)
  dx = x - cx
  dy = y - cy
  (cx + dx*cos(angle) - dy*sin(angle),
   cy + dx*sin(angle) + dy*cos(angle))
```

Rotate each position around the center. Angle is in radians; positive angles turn clockwise with screen coordinates. Apply the same mapping to each point of geometry.

Objects are items with positions, such as glyphs, points, or images.

```text
scroll(objects,dx,dy): move each object's position from (x,y) to (x+dx,y+dy)
```

Positive offsets move right and down; negative offsets move left and up.

```text
wave(objects,axis,amplitude,phase):
  add amplitude*sin(phase) to each object's position along axis
```

Axis is x (horizontal) or y (vertical). Amplitude is the maximum distance from the starting position; phase is an angle in radians. Apply wave once per axis to move along both axes.

```text
perspective(points,center,focal):
  center = (cx,cy)
  map each (x,y,z) to (cx + focal*x/z, cy + focal*y/z)
```

Coordinates are relative to a camera at the origin, looking along positive z; z must be positive. Center is the screen position of the viewing axis. Focal is the focal length in screen pixels; larger values spread points farther from the center. Retain each point's original z for depth coloring.

```text
bitand((x,y)): floor(x) & floor(y)
```

`&` compares integer bits: a result bit is 1 only when both input bits are 1. DEN includes flooring the coordinates in this operation.

### State and arithmetic

```text
bin(value,boundaries): count the boundaries less than or equal to value
```

Boundaries must be numbers in strictly increasing order. The result is the zero-based range index; a value equal to a boundary belongs to the higher range.

```text
lookup(index,values): return the value at index in values, counting from 0
```

Index must be an integer from 0 through length(values)-1.

```text
next(start): start on the first frame, increasing by 1 each frame without an upper limit
```

A named counter is evaluated once per frame. Every reference to that name uses the same value for that frame.

```text
prev(name,steps): named value from steps updates earlier
```

Steps is a positive integer, defaulting to 1.

`blur` averages a selected neighborhood. Orthogonal means left, right, up, and down, excluding diagonals. In these formulas, directions name neighbors rather than whole edges.

```text
blur(4): (left + right + up + down) / 4
blur(5): (center + left + right + up + down) / 5
blur(8): (left + up_left + up + up_right + right + down_right + down + down_left) / 8
blur(9): (center + left + up_left + up + up_right + right + down_right + down + down_left) / 9
```

Only 4, 5, 8, and 9 are defined. At edges, average available cells; if none are available, leave the value unchanged.

```text
floor(value): greatest integer less than or equal to value
add(value,amount): value + amount
mul(value,factor): value * factor
sub(value,amount): value - amount
sub((x,y),(cx,cy)): (x-cx,y-cy)
div(value,divisor): value / divisor
mod(value,divisor): value % divisor
perturb(value,min,max): add(value,random(min,max))
damp(value,factor): mul(value,factor)
decay(value,amount): sub(value,amount)
```

Divisors must be nonzero. `%` is nonnegative modulo for positive divisors. Damp's factor is between 0 and 1; decay's amount is nonnegative. Bare perturb means add(random), changing values rather than positions.

Arithmetic also operates on a selected axis of points:

```text
add(points,axis,amount)
sub(points,axis,amount)
mul(points,axis,factor)
div(points,axis,divisor)
mod(points,axis,divisor)
```

Apply the scalar operation to the selected coordinate of every point, leaving other coordinates unchanged. Axis is x, y, or z and must exist in the point's dimensions. The same divisor rules apply.

`random(min,max)` supplies a uniformly chosen integer, including both bounds. Bounds must be ordered integers. Draw separately at each location and occurrence. Bare `random` leaves range and distribution unspecified.

```text
select(items,condition): select the items satisfying condition
move(items,position): move every item to position
move(items,selection,position): move only selected items to position
clamp(value,min,max): min if value < min; max if value > max; value otherwise
clamp(value,min): max(value,min)
wrap(value,min,max): min + ((value - min) % (max - min))
wrap(objects,axis,min,max):
  wrap each object's selected coordinate; leave the other unchanged
```

- `select`: evaluate condition separately for each item using its current coordinates. The selection identifies items from its input for later operations.
- `move`: evaluate the destination position separately for each moved item; x, y, and z refer to its current position. Field values travel to their destinations; points change position. Preserve what moves and leave unselected items unchanged. Explain field edges, overlapping writes, and unwritten locations beside the effect.
- `clamp`: min must be at most max; both bounds are included.
- `wrap`: min must be below max; the result includes min and excludes max. For objects, axis is x or y and the selected coordinate uses the scalar wrap formula.

### Composition and output

```text
over(foreground,background):
  use the foreground pixel where opaque; otherwise use the background pixel
```

Transparency is separate from color indices. Index 0 may be opaque.

```text
draw(elements,mark): render each element at its position using its color index
```

The result is an indexed image. Mark supplies a shape for points; when omitted, use the element's existing shape, such as a glyph. Explain size, background, and overlap beside the effect.

```text
color(elements,index): assign index as each element's color index
```

Evaluate index separately for each element using its coordinates. Preserve its position and other attributes. After perspective, z refers to the retained original depth.

```text
inject(destination,source):
  source(x,y) if defined(source(x,y)) else destination(x,y)
palette(index,colors): colors[index]
```

`defined` means the source supplies a value, including zero. Palette colors may be externally supplied; selection controls are outside DEN.

```text
raster(colors,index,rgb):
  for each screen row y, replace palette entry index with rgb
```

Evaluate rgb separately for each row and preserve other entries. RGB components are integers from 0 through 255. The result is a palette that varies by row; palette uses the colors for the pixel's row. Pixel indices remain unchanged.

## Effects

These sketches describe the current Lua effects. Detailed implementation checks are retained in the workflow's source review.

### Cloud

```text
cloud: prev(cloud) → blur(5) → perturb(-1,1) → wrap(0,16)
image: cloud → floor → palette
```

Initialize randomly. Smooth and perturb the field, wrap intensities, and floor values for rendering.

### Fire

```text
source: seed(bottom,random(12,15))
fire: prev(fire) → inject(source) → move((x+random(-1,1),y+random(-3,1))) → decay(random(0,1)) → clamp(0)
image: caption → over(fire) → palette
```

Start from the framebuffer. Refresh the bottom, scatter values mostly upward, and reduce intensity. Clamp destination coordinates to the screen. Writes occur in place, so traversal and overlaps affect the result.

Caption is a supplied indexed image of "the fire rises" in color index 5, transparent outside the letters and entirely transparent when hidden. The source draws proportional text at scale 2, at ((w-length(content)*11)//2,h//2), when t%320<60 after incrementing t. It draws directly into the framebuffer, so caption pixels can influence the next fire update; the overlay chain shows composition, while the source retains this feedback.

### Roto

```text
t: next(0)
angle: t/45
center: (120*(1-sin(angle)),68*(1-sin(angle)))
uv: screen → sub(center) → rotate((0,0),angle)
indices: uv → bitand → div(5)
image: indices → palette
```

Subtract the moving center, then rotate about the origin to obtain pattern coordinates. Divide the AND result by 5. Conversion to a discrete palette index is left to the renderer and remains unspecified.

### Ripple

```text
source: seed(drop_region,512)
level: prev(level) → blur(4) → mul(2) → sub(prev(level,2)) → damp(0.98) → inject(source)
texture: screen → linear
uv: screen → radial(center,1 - level/1024)
image: uv → sample(texture) → palette
```

Initialize both level histories at zero (rest). Update only locations with all required neighbors. The caller supplies a 5×5 drop region from the pointer or a random position; no drop means an empty region.

The texture uses `a=(1-1/240)/8`, `b=-1/8`, and `c=0`. Center is the screen center. Floor scaled offsets before adding center, then clamp sampling coordinates to texture bounds. Injection happens after rendering, so new drops appear later.

### Wobble

```text
t: next(0)
letters: text("Wobble! Wobble!",(i*18,60)) → scroll(-t,0) → wrap(x,0,312) → wave(y,10,i+t/9)
image: letters → color(12) → draw → palette
```

Here, `i` is the glyph index, starting at 1, and also supplies wave's phase. Recompute positions from the text's initial layout each frame. The wrap width 312 is screen width 240 plus four glyph spacings of 18. Draw fixed-width glyphs at scale 3 on a background of index 0.

### Starfield

```text
initial: points((random(-120,120),random(-67,67),random(120,240)),64)
moved: prev(stars) → sub(z,1)
near: moved → select(z < 120)
stars: moved → move(near,(random(-120,120),random(-67,67),240))
image: stars
  → perspective((120,68),119)
  → color(lookup(bin(z,(160,200)),(15,8,3)))
  → draw(squares)
  → palette
```

Initialize stars once from `initial` before frame updates. Stars at `z=120` remain visible for that frame. Draw 2×2-pixel squares on a background of index 0; later stars overwrite earlier stars where their marks overlap.

### Copper

```text
t: next(1)
shape: seed(screen,0) → inject(seed(band,1))
intensity: floor((sin(y*0.25+t*0.08)+1)*127.5)
scan: base_palette → raster(1,(intensity,floor(intensity*0.35),floor(intensity*0.05)))
image: shape → palette(scan)
```

The counter starts at 1, matching the source increment before scanline rendering. `band` is a supplied region covering every screen column and rows 44 through 91 inclusive. `base_palette` is a supplied palette.
