---
name: hildegard-openscad
description: Use when the user says OpenSCAD or .scad, or wants an exact parametric part (bracket, mount, box, enclosure, snap fit, clip, thread, knob, gear) - code Hildegard renders, measures and draws.
---

# Parts in OpenSCAD

OpenSCAD is code-CAD: the model is a short program of solids, booleans and transforms, so every dimension is a named
number and every round is a small edit you can say in a sentence. That makes it the strongest route for an agent to
make a part that has to fit, and the weakest for anything organic. Sculpt figures and animals with **hildegard-3d**'s
signed-distance route or Blender; build parts here.

This skill works alone. With Hildegard linked, it renders your `.scad`, measures it and draws it each round
(`loop_submit`). Without Hildegard, you render with the `openscad` program yourself and measure the STL with
`scripts/review_mesh.py` (Python with numpy and Pillow: `python3 -m pip install pillow numpy av`); section 7 has the commands. Without them,
read the size from OpenSCAD's own render and look at it. If you have them, the
**hildegard-3d** skill covers the brief, the sheet and the checks, and **hildegard-print** (for a part that has to
work) the evidence for every dimension and fits from the printer's clearance.

## 1. What this computer has

With Hildegard linked, call `openscad_status {}`. Without it, run `openscad --version` and look in OpenSCAD's
libraries folder for BOSL2 (Linux `~/.local/share/OpenSCAD/libraries`, macOS
`~/Documents/OpenSCAD/libraries`, Windows `Documents\OpenSCAD\libraries`). If `openscad` is not found, tell your
user how to install it (openscad.org), or write the `.scad` and say it is unrendered. `openscad_status` says the
OpenSCAD version, whether it renders with **Manifold** (booleans in seconds) or **CGAL** (2021.01, which
can take minutes on the same model), and which libraries `include <…>` will find. Write for what is there:

- **BOSL2 installed**: use it (section 4). It rounds, chamfers, attaches and threads in one line each.
- **No BOSL2**: plain OpenSCAD. Don't include a library that isn't listed; the render fails.
- **CGAL only**: keep `$fn` at 24 to 32 while blocking out, raise it to 64 to 96 for the final round, and avoid
  `minkowski()`.
- **Not installed**: tell your user how (the tool says), or export STL yourself and send that.
- **Not sandboxed** (Windows, Linux without bubblewrap, or an AppImage; the tool says): a round may be only `.scad`
  files of its own. No libraries, no `import()` or `surface()` at all, and `linear_extrude` and `rotate_extrude` take
  their height by name (`height = h`), not by position. Send the library's files inside the round, beside the `.scad`
  that includes them, or write plain OpenSCAD.

## 2. The file

One file, `main.scad`, in this order. Every number the part is built from is a named parameter at the top, with its
unit and where it came from; the geometry holds no bare numbers but 0, 1, 2 and the small `eps`.

```openscad
// Cable clip for a 6 mm cable, desk edge 18 mm. Millimetres, Z up, standing on z = 0.

/* [Fit] */
cable_d    = 6.0;   // measured, calipers
grip       = 0.3;   // squeeze on the cable, assumed
desk       = 18.0;  // user said
/* [Build] */
wall       = 2.4;   // six perimeters at 0.4 mm
width      = 12;
$fn        = $preview ? 32 : 96;
eps        = 0.01;  // overlap so cuts don't leave skins

slot_d = cable_d - grip;

difference() {
    cube([desk + 2 * wall, width, 20]);
    translate([wall, -eps, -eps]) cube([desk, width + 2 * eps, 20 - wall + eps]);
    translate([-eps, width / 2, 20 + slot_d / 4]) rotate([0, 90, 0]) cylinder(d = slot_d, h = wall + 2 * eps);
}
```

- **Units are millimetres, Z up, the part standing on z = 0** in the pose it prints in. Hildegard's `base` and
  `overhang` checks read it that way.
- **Customizer comments** (`/* [Group] */`, `// note`) make the parameters editable in OpenSCAD's own panel, so your
  user can change the part without you.
- **`eps` overlaps on every cut.** A cut whose face lands exactly on a face leaves a paper-thin skin or a
  non-manifold edge. Make the cutter longer by `eps` on each open side.
- **Echo what you need to check**: `echo(slot = slot_d, inner = desk);`. Hildegard returns every echo with the round,
  so you can read the numbers you built from beside the measurements.

## 3. Habits that make good parts

- **2D first, then extrude.** Most parts are a profile: draw it with `square`, `circle`, `polygon` and `offset`, then
  `linear_extrude` or `rotate_extrude`. `offset(r = 2) offset(delta = -2)` rounds every outside corner of a profile;
  `offset(r = -2) offset(delta = 2)` rounds the inside ones.
- **`hull()` for organic-looking hard surfaces**: two cylinders hulled make a slot or a rounded bar; four spheres make a
  rounded block. Cheap and exact.
- **Modules for repeated features**, with the dimensions as arguments: `module screw_post(d, h) { … }`. A `for` loop
  places them; never copy and paste a feature.
- **Holes for real hardware**: an M3 clearance hole is 3.4 mm, an M3 heat-set insert hole about 4.0 mm, an M3 nut trap
  5.7 mm across the flats (`cylinder(d = 5.7 / cos(30), $fn = 6)`). Check the insert's own sheet when your user names
  one, and say which numbers are assumed.
- **Horizontal holes print oval at the top.** Use a teardrop (a `hull()` of the circle and a point above it at 45°)
  or a flat top, and the part needs no supports.
- **Chamfer the bottom edge** 0.4 to 0.6 mm so elephant's foot doesn't stop a part fitting; round the top edges for
  the hand.
- **Text**: `linear_extrude(0.6) text("A1", size = 6, font = "Liberation Sans:style=Bold", halign = "center");`. At least
  5 mm tall and 0.6 mm deep to print legibly.
- **Expensive operations**: `minkowski()` with a sphere and big `$fn` costs minutes (seconds with Manifold);
  `hull()` over many children and huge `for` unions do too. Round with `offset` in 2D or BOSL2's `rounding=` instead.

## 4. BOSL2, when it's there

`include <BOSL2/std.scad>` once at the top. What it saves you:

| Need | BOSL2 |
| --- | --- |
| A rounded or chamfered block | `cuboid([40, 30, 10], rounding = 2, edges = "Z", anchor = BOTTOM)` |
| Round only some edges | `cuboid(s, rounding = 2, except = BOTTOM)`, `edges = [TOP, "Z"]` |
| A rounded cylinder or tube | `cyl(h = 10, d = 20, rounding2 = 1)`, `tube(h = 10, od = 20, id = 16)` |
| Put one part on another | `cuboid(s) attach(TOP) cyl(h = 5, d = 8, anchor = BOTTOM);` |
| Cut with tags instead of `difference()` | `diff() cuboid(s) tag("remove") attach(TOP) cyl(h = 4, d = 3.4, anchor = TOP);` |
| Rows and grids | `xcopies(spacing = 10, n = 4)`, `grid_copies(spacing = 12, n = [3, 2])` |
| Threads | `include <BOSL2/threading.scad>` then `threaded_rod(d = 10, l = 20, pitch = 1.5)` |
| Screws and their holes | `include <BOSL2/screws.scad>` then `screw("M3", length = 10)`, `screw_hole("M3", length = 10, head = "socket")` |
| Gears | `include <BOSL2/gears.scad>` then `spur_gear(mod = 1, teeth = 20, thickness = 6)` |
| Text on a face | `text3d("A1", h = 0.6, size = 6, anchor = BOTTOM)` |

Anchors (`TOP`, `BOTTOM`, `LEFT`, `FRONT`, `CENTER`) place a shape by a named point instead of by arithmetic, which is
where most hand-written OpenSCAD goes wrong. Use `anchor = BOTTOM` on the base so the part stands on z = 0. For a
printed thread, give it the printer's clearance: `threaded_rod(…, $slop = 0.1)` (BOSL2's `$slop` is clearance per
side; take it from `printer_profile`, or from your user's printer notes).

## 5. Parts that fit together

For a print with more than one part (a box and its lid, a pin and its sleeve), declare the parts at the top level and
draw each one when `part` names it. Hildegard renders every part on its own and measures the fits between them:

```openscad
parts = ["box", "lid"];   // Hildegard renders each with part set to its name
part  = "all";
fit   = 0.2;              // slide clearance per side, from the printer's profile

module box() { … }
module lid() translate([0, 0, box_h]) { … }   // where it sits assembled

if (part == "all" || part == "box") box();
if (part == "all" || part == "lid") lid();
```

Model the parts **assembled**, where they sit together; say how each pair mates in the loop's `fits`
(`{"a": "lid", "b": "box", "kind": "slide"}`) and how each prints in `print_pose`. Size every mating feature from the
printer's clearance, never a guessed number: a 20 mm boss in a slide fit gets a hole of `20 + 2 * fit`.

## 6. Enclosures, snap fits, clips and threads

What decides whether a case around a board works, and what each round checks:

- **The board's own drawing**, named with its revision. Boards of one outline move their ports between versions: the Pi 4 and Pi 5 swap
  Ethernet and USB. On the Pi 5 (its drawing, from the corner by USB-C): USB-C 11.2, micro HDMI 25.8 and 39.2 along the long edge; Ethernet
  10.2, USB 29.1 and 47 up the short edge, sockets 3 mm past it; power button 18.4 and microSD about 28, under the board, on the other one;
  GPIO pins 8.5 high along the back. The Pi 4 has USB 9 and 27, Ethernet 45.75. Positions from memory are marked `assumed` and checked with
  a paper print at 1:1 before any plastic.
- **Every opening has a line**: each port, the card slot, the button, the header and the camera ribbon is cut, or said to be closed and why.
  Say what holds the board down (screws in the posts) and what lines up the lid (a rim), or the snaps do both jobs badly.
- **A cutout passes the plug's body, not the socket.** Typical overmoulds: USB-C 13 x 7.5 mm, micro HDMI 11 x 8, HDMI 22 x 12, barrel jack
  10 round, RJ45 and USB-A stacks open to the top,
  or a board whose sockets overhang can't be lowered in. Measure the cable that will be used. An opening smaller than the plug's body needs the
  socket's face within 1 mm of the wall's outside, or the plug can't seat: thin the wall there.
- **A snap is a spring.** Strain at the root is `1.5 * thickness * hook / length^2`, length being only what bends: 2 percent for PLA, 4 for PETG, 6 for ABS, half
  that when it opens often. Print the arm lying along its length; layers across the root snap off. One list places the hooks and their
  windows, so the halves can't disagree. A lid printed outside face down has its arms standing up, across the layers: half those figures again
  (PLA 1 percent, PETG 2), and a longer arm.
- **Air**: an inlet low and an outlet high, slots at most 3 mm wide (a fingertip and a screw stay out), the outlet over the hottest chip.
- **A clash render with a control.** Intersect the case with a ghost of the board and its plugs, placed from the drawing and never from the list that
  cuts the openings (a missing or moved opening would take its plug along and pass): it must be empty. Then break it on purpose
  (`probe = true`, or delete an opening): it must not be. A check that can't fail has not checked anything. OpenSCAD says "empty" after a failed
  `assert` as well, so a result counts only when no ERROR or WARNING line came with it.
- **Checks your user runs stand alone.** They have the `.scad` and OpenSCAD, not this skill's scripts: give `openscad` commands with the
  answer each must print, or the whole script, and name no file or program they were not given. Numbers you expect come from an `echo` in the file, never from your head: a mouth cut or a
  rounded tip moves the outside size too.

```openscad
part  = "all";      // all, base, lid, lid_print; checks: clash, drop, mate, pull
parts = ["base", "lid"];   // the printed pieces, each rendered on its own
probe = false;      // true: the ghost goes 1 mm into the left wall, so clash must report a solid (the negative control)
fit = 0.2; wall = 2.4; floor_t = 2; lid_t = 2; over = 1.5; clr = over + fit; eps = 0.01; gap = 0.02; $fn = $preview ? 32 : 64;      // over: how far the front sockets stand past the board (assumed, measure yours); gap: a real one between parts that rest on each other

/* [Board] Raspberry Pi 5, from its maker's drawing: what is there. Nothing below is cut from this list, so a wrong cut shows */
board = [85, 56, 1.4]; holes = [[3.5, 3.5], [61.5, 3.5], [3.5, 52.5], [61.5, 52.5]]; hole_d = 2.7;
post_h = 5; post_d = 5.5; pilot_d = 2.1; tall = 20;               // M2.5 screws hold the board; tall: highest part above it, plus air
up = [[7, 50, 51, 5, 8.5], [66, 2.2, 22, 16, 13.5], [71, 21.3, 17, 33.5, 16],              // x, y, width, depth, height: GPIO, Ethernet, both USB stacks
      [6.7, -over, 9, 7, 3.2], [22.55, -over, 6.5, 7, 2.9], [35.95, -over, 6.5, 7, 2.9]];  // USB-C, micro HDMI
plugs = [["front", 11.2, [13, 7.5], 1.6], ["front", 25.8, [11, 8], 1.45], ["front", 39.2, [11, 8], 1.45],      // side, center, body w x h, center above the board: USB-C, micro HDMI
         ["right", 10.2, [12, 14], 6.75], ["right", 38.05, [33, 16], 8], ["left", 28, [11, 1], -2], ["left", 18.4, [3, 3], 1.25]];   // Ethernet, USB, microSD card, a pen tip on the button
/* [Openings and posts] side, center, from, to above the board, width. Up to 99: open to the rim, and the lid's skirt closes what is left above the socket */
open = [["front", 11.2, -2.4, 5.6, 13.5], ["front", 25.8, -2.8, 5.7, 11.5], ["front", 39.2, -2.8, 5.7, 11.5],
        ["right", 10.2, -0.4, 99, 17], ["right", 38.05, -0.4, 99, 34.5], ["left", 28, -5, 0, 12], ["left", 18.4, -0.75, 3.25, 4.6]];
posts = [[3.5, 3.5], [61.5, 3.5], [3.5, 52.5], [61.5, 52.5]];
skirt = [["right", 10.2, 14.5, 17], ["right", 38.05, 17, 34.5]];                           // side, center, from above the board, width of the opening it fills
// closed on purpose: GPIO, camera and display ribbons, PCIe, the fan plug (all inside); say so to your user
/* [Snaps] side, center: both halves read this one list */
snaps = [["front", 62], ["back", 64], ["left", 8], ["left", 48]]; rim_h = 1.5; rim_t = 1.2;
arm_l = 16; arm_t = 1.5; arm_w = 6; hook = 0.6; hook_h = 2; play = 0.3; strain_max = 0.02;     // arms stand up in the print: PETG 2 %, PLA 1 %
vent = [2.4, 14]; vent_at = [28, 22]; vent_n = 6; vent_p = 5;                                 // outlet in the lid over the chip: slot w x length, first, how many, pitch
inlet = [2.4, 8]; inlet_at = 8; inlet_n = 10;                // slots low in the back wall, clear of a desk: w x height

in_l = board[0] + 2 * clr; in_w = board[1] + 2 * clr; top = floor_t + post_h + board[2]; h = top + tall;
free = arm_l - rim_h - hook_h / 2;                          // the rim holds the top of the arm; what bends is below it
strain = 1.5 * arm_t * (hook + fit) / pow(free, 2);         // straight rectangular cantilever
echo(outer = [in_l + 2 * wall, in_w + 2 * wall, h + gap + lid_t], strain = strain);
assert(strain <= strain_max, "snap too stiff: longer or thinner arm, or a smaller hook");
assert(hook >= 2 * fit, "hook too small to hold");
assert(gap > 0, "a lid resting on the base merges with it");
assert(len([for (p = ["all", "base", "lid", "lid_print", "clash", "drop", "mate", "pull"]) if (p == part) p]) == 1, str("no part named ", part));     // an unknown one is empty, which would pass
assert(post_h >= 3, "no room for solder joints under the board");
assert(h - arm_l > top, "a snap arm reaches the board");
for (s = skirt) assert(top + s[2] < h, "a skirt above the rim");

module at(side, x) {                                        // +y points out of the wall, y = 0 on its inner face
    assert(len([for (s = ["front", "back", "left", "right"]) if (s == side) s]) == 1, str("no side named ", side));
    if (side == "front") translate([x, -clr, 0]) rotate(180) children();
    if (side == "back") translate([x, board[1] + clr, 0]) children();
    if (side == "right") translate([board[0] + clr, x, 0]) rotate(-90) children();
    if (side == "left") translate([-clr, x, 0]) rotate(90) children();
}
module box(w, d, z0, z1) translate([-w / 2, 0, z0]) cube([w, d, z1 - z0]);
module shell(z) translate([-clr - wall, -clr - wall, 0]) cube([in_l + 2 * wall, in_w + 2 * wall, z]);
module slots(z, t) for (i = [0 : vent_n - 1]) translate([vent_at[0] + i * vent_p, vent_at[1], z - eps]) cube([vent[0], vent[1], t + 2 * eps]);
module base() difference() {
    union() {
        difference() { shell(h); translate([-clr, -clr, floor_t]) cube([in_l, in_w, h]); }
        for (p = posts) translate([p[0], p[1], floor_t - eps]) cylinder(d = post_d, h = post_h + eps);
    }
    for (p = posts) translate([p[0], p[1], floor_t + 1]) cylinder(d = pilot_d, h = post_h);
    for (o = open) at(o[0], o[1]) translate([0, -eps, 0]) box(o[4], wall + 2 * eps, top + o[2], min(top + o[3], h + eps));      // the plug's body must pass, not only its tip
    for (s = snaps) at(s[0], s[1]) translate([0, -eps, 0]) box(arm_w + 2 * play, wall + 2 * eps, h - arm_l - play, h - arm_l + hook_h + play);
    for (i = [0 : inlet_n - 1]) at("back", inlet_at + i * vent_p) translate([0, -eps, 0]) box(inlet[0], wall + 2 * eps, floor_t + 1, floor_t + 1 + inlet[1]);
}
module lid() difference() {
    union() {
        translate([0, 0, h + gap]) shell(lid_t);
        translate([0, 0, h - rim_h]) linear_extrude(height = rim_h + gap + eps) difference() {                        // rim: lines the lid up
            translate([-clr + fit, -clr + fit]) square([in_l - 2 * fit, in_w - 2 * fit]);
            translate([-clr + fit + rim_t, -clr + fit + rim_t]) square([in_l - 2 * (fit + rim_t), in_w - 2 * (fit + rim_t)]); }
        for (s = skirt) at(s[0], s[1]) translate([0, -fit - eps, 0]) box(s[3] - 2 * fit, wall + fit + eps, top + s[2], h + gap + eps);      // the tall sockets go in from above
        for (s = snaps) at(s[0], s[1]) {
            translate([0, -fit - arm_t, 0]) box(arm_w, arm_t, h - arm_l, h + gap + eps);
            translate([-arm_w / 2, 0, h - arm_l]) rotate([90, 0, 90]) linear_extrude(height = arm_w)
                polygon([[-fit - eps, 0], [hook, hook_h / 2], [hook, hook_h * 0.7], [-fit - eps, hook_h]]);
        }
    }
    slots(h + gap, lid_t);
}
module board_ghost(rise = 0) translate([probe ? -clr - 1 : 0, 0, floor_t + post_h + gap]) {
    difference() { cube(board + [0, 0, rise]); for (p = holes) translate([p[0], p[1], -eps]) cylinder(d = hole_d, h = board[2] + rise + 2 * eps); }
    for (u = up) translate([u[0], u[1], board[2]]) cube([u[2], u[3], u[4] + rise]);
}
module screw_ghost() for (p = holes) translate([p[0], p[1], floor_t + 1 + gap]) cylinder(d = pilot_d - 0.2, h = post_h);
module plug_ghost() for (p = plugs) at(p[0], p[1]) translate([0, gap, 0]) box(p[2][0], 20, top + p[3] - p[2][1] / 2, top + p[3] + p[2][1] / 2);

if (part == "all" || part == "base") base();
if (part == "all" || part == "lid") lid();
if (part == "lid_print") translate([0, 0, h + gap + lid_t]) rotate([180, 0, 0]) lid();       // outside face on the bed
if (part == "mate") intersection() { base(); lid(); }
if (part == "pull") intersection() { base(); translate([0, 0, 1]) lid(); }                  // lifted 1 mm, a hook must catch
if (part == "clash") intersection() { union() { base(); lid(); } union() { board_ghost(); plug_ghost(); screw_ghost(); } }
if (part == "drop") intersection() { base(); board_ghost(h); }              // the board goes straight down into place
```

```sh
# save the program above as case.scad in your working folder first; -o wants a name ending in .stl, /dev/null is refused. A variable used before it is set is a warning, so make it fatal
# "empty" below: it prints "Current top level object is empty" and no ERROR or WARNING line. "a solid": none of those lines, and the .stl is written
openscad --hardwarnings -o base.stl -D 'part="base"' case.scad                      # want: a solid
openscad --hardwarnings -o lid.stl -D 'part="lid_print"' case.scad                  # want: a solid
openscad --hardwarnings -o clash.stl -D 'part="clash"' case.scad                    # board and plugs against the case, want: empty
openscad --hardwarnings -o probe.stl -D 'part="clash"' -D probe=true case.scad      # the board 1 mm into a wall, want: a solid, or the check is dead
openscad --hardwarnings -o probe2.stl -D 'part="clash"' -D 'open=[]' -D 'up=[]' case.scad  # only plugs, no openings, want: a solid
openscad --hardwarnings -o drop.stl -D 'part="drop"' case.scad                      # the board's way in, want: empty; with -D clr=0.5: a solid
openscad --hardwarnings -o mate.stl -D 'part="mate"' case.scad                      # lid against base, want: empty; with -D fit=-0.3: a solid
openscad --hardwarnings -o pull.stl -D 'part="pull"' case.scad                      # lid lifted 1 mm, want: a solid (at least one hook overlaps), or nothing latches
```

**A clip that grips a pipe or hose** can't be checked by a clash with the part seated: that only tests the bore. Measure the bore and the
mouth with gauge pins, as a machinist would: the go pin passes (empty), the no-go pin doesn't (a solid). Round or flare the lips outward
from the mouth, never into it. Print it on the end of its ring as one constant outline (`linear_extrude`), so no lip starts in mid-air; when
the desk slot and the ring face different ways, one of them has that lip, and it needs a 45 degree ramp down to the wall or two parts. Keep the screw heads where a driver reaches them straight on with the ring in the way.

```openscad
R = (bore + wall) / 2;  phi = 180 - asin(mouth / bore);    // the arm is an arc from the back of the ring to the lip
strain = wall * ((body - mouth) / 2) / (2 * R * R * (phi * PI / 360 - sin(2 * phi) / 4));      // pushing the lips open by the body
assert(strain <= strain_max, "clip too stiff: thinner wall or wider mouth");                   // 0.02: PETG opened often (0.04 if fitted once, PLA half); a straight-arm formula misreads a ring
module gauge(d, reach) translate([0, 0, 1]) linear_extrude(height = height - 2) hull() { circle(d = d); translate([0, reach]) circle(d = d); }
if (part == "bore")  intersection() { clip(); gauge(pin, 0); }      // pin = bore - 0.2: want empty; bore + 0.2: want a solid
if (part == "mouth") intersection() { clip(); gauge(pin, bore); }   // pin = mouth - 0.1: want empty; mouth + 0.2: want a solid
```

**A threaded lid** is turned and lifted together in its clash check, `translate([0, 0, pitch * a / 360]) rotate(a) lid();`, or a good
thread fails at every angle but one. Start seated and use `a` of 0 and up, which unscrews a right-hand thread (below 0 drives it into the rim);
a left-hand one turns by `-a`. A good part that clashes at 0 has a phase error: turn one of the two threads by a fixed angle.

Print one corner with one snap and one port first (20 minutes), then the case. The comments in a `.scad` you hand over name the part, its
numbers and where they came from, never the tool that made it.

## 7. The loop

Without Hildegard, each round is: edit `main.scad`, render it, measure it, look at the sheet, change one thing.

```sh
openscad -o round-1.stl main.scad            # add --backend=Manifold when this OpenSCAD has it (much faster)
echo '{"path": "round-1.stl", "sheet": "round-1.png", "print": true, "size_mm": [60, 30, 20]}' > spec.json
python3 scripts/review_mesh.py spec.json
```

It prints the mesh checks as JSON (watertight, manifold, size, base, overhang and the rest) and writes the sheet, eight
views and silhouettes, to `round-1.png`: open that image and look at it. Add `bed_mm` from the printer. For a part
with several pieces, render each alone (`-D 'part="lid"'`) and check each; the checker measures fits only between named
parts of one OBJ, GLB or ASCII STL, so otherwise check a fit by the numbers in the file. Keep one line per round in
`rounds.md`. Warnings OpenSCAD prints (`Ignoring unknown variable`) are failures: fix every one.

With Hildegard linked, start it as **hildegard-3d** says (`loop_start` with kind `model`, `print: true` for a print),
then send the source itself:

- **One file**: `loop_submit {"text": "<the source>"}`, or `file_path` to the `.scad`.
- **Several files** (main.scad with `use <parts/hinge.scad>`, an SVG to import): `file_path` to the folder, or `data`
  as a base64 ZIP. The file rendered is `main.scad`.

Hildegard renders it with OpenSCAD and checks the mesh like any other round: eight views, silhouettes, watertight,
size, base, overhangs, the fits. The round keeps your source, so the owner can open it.

What Hildegard refuses, before anything runs (no round is used, so fix it and send again):

- `include` or `use` of anything but a `.scad` file (lowercase), of an absolute path, or of one with `..`. Paths are
  read from the folder of the file that includes them: the round's own files by that path, and libraries by their own
  (`include <BOSL2/std.scad>`), only where OpenSCAD runs sandboxed. More than 4 MB of source, or brackets nested
  deeper than 256, are refused too.
- `include` or `use` written any other way than `include <file.scad>`, such as with a comment before the `<`.
- `import()` or `surface()` with anything but one plain string naming a file sent with the round, from the folder of
  the file that reads it. Send the SVG or STL beside the `.scad` that imports it. Its other strings (a DXF layer, an
  SVG id) can't look like paths. Data files are read only where OpenSCAD runs sandboxed.
- `import_stl`, `import_off`, `import_dxf`, any `dxf_…`, `file = …` on any call but `import()` and `surface()`, and a
  string given by position to `linear_extrude` or `rotate_extrude` (that's a file to them).
- Files or folders in the round whose names start with a dot.

OpenSCAD then runs sandboxed where it can be (sandbox-exec on macOS, bubblewrap on Linux): no network; it reads only
the system, its own files, fonts, the libraries and the round; and it writes only in the round. It gets no
environment beyond what it needs, and one render runs at a time on the computer (a few more wait; past that, "busy":
send again in a few minutes). The whole round, every part of it, shares one time limit, counted while OpenSCAD runs
(180 s, or `HILDEGARD_OPENSCAD_TIMEOUT` seconds). Paths in OpenSCAD's messages come back as round-relative names,
`<library>/…` or `<path>`. A mesh over 256 MB, as STL or as the parts' OBJ, is refused: lower `$fn`.

What fails at render, with OpenSCAD's own message (no round is used):

| It says | It means |
| --- | --- |
| `Parser error … line N` | A missing `;`, `)` or `}` on or just before line N |
| `The model is empty` | A module defined and never called, or a `difference()` whose first child is empty |
| `The model is 2D` | A profile never extruded |
| `took longer than` | `$fn` too high, `minkowski()`, or a huge `for` union; block out lower |

And **warnings come back as a check** (`openscad` warns). `Ignoring unknown variable` or `unknown module` means a part
is missing from the model while the render looks fine: fix every one before you answer the rubric.

## 8. Things that go wrong

- **Faces exactly coincident.** A lid resting on a box at the same z, a cutter flush with a face: Z-fighting in the
  preview, skins and non-manifold edges in the mesh (`manifold` and `debris` catch them). Overlap by `eps`, or keep a
  real gap between separate parts.
- **Thin walls from subtraction.** Two cuts that come within a millimetre leave a wall the printer can't make. Echo the
  wall you meant (`echo(wall = outer - inner)`) and keep it above 1.2 mm for FDM.
- **The part made at the origin's corner but meant centred**, or the other way. `cube(s, center = true)` and BOSL2's
  anchors say which; pick one and keep it for the whole file.
- **A preview that lies.** `$preview` geometry and `%` or `#` modifiers don't export. Hildegard checks the rendered
  mesh, not the preview; trust the sheet.
- **Polygon winding and `polyhedron()`.** Faces must list points clockwise seen from outside; a wrong one shows magenta
  on the sheet. Prefer `hull()` or `linear_extrude` to hand-written polyhedra.
- **Detail before the masses are right.** Get the outline, the fits and the size passing first; text, knurling and
  fillets come last.

## 9. Deliver

The finished round's file and its `main.scad` (with the parameters your user can change and where each came from),
the sheet, and for a print the print sheet (`loop_finish` writes it; without Hildegard, write it yourself, as the
**hildegard-print** skill describes). Say which numbers were measured and which assumed.
No weapons or weapon parts, in any form.
