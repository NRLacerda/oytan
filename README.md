# Oytan — 2D Math & Engineering Renderer

## Why the name?

**Oytan** fuses two words from *Tupi* — the indigenous language of Brazil's first peoples.
*Oby* means both green and blue. *Pytã* means red.
Together they form the RGB triad: the atomic unit of every image this engine produces.

No magic. No GPU pipeline. Just pixels — and the mathematics that place them.

---

## What Oytan is

Oytan is a **C software renderer** built from scratch for mathematical visualization and engineering simulation. It starts with a flat array of pixels in memory and builds upward through geometry, coordinate systems, function plotting, and physical simulation.

The focus is **seeing mathematics**:

- Plot `f(x)` and watch Bhaskara's roots appear as intersections on the curve
- Render a vector field and see a force gradient across a plane
- Launch a teleguided missile and trace its proportional-navigation path to a moving target
- Annotate everything with dimension lines, tick labels, and HUD readouts

If this later becomes a foundation for a game engine, that's a natural consequence — not the goal.

---

## Architecture

Each layer depends strictly on the one below it. There are no shortcuts.

```
Framebuffer  (raw pixel memory)
      ↓
drawPixel()  (atomic write with bounds + clip check)
      ↓
Primitives   (Bresenham lines, filled shapes, Bézier curves)
      ↓
Math renderer  (axes, function plots, parametric/polar curves, vector fields)
      ↓
Transforms + Camera  (world ↔ screen, zoom/pan, Mat3 pipeline)
      ↓
Render pipeline  (layered queue, blend modes, clip stack, text)
      ↓
Simulation  (kinematics, guidance law, moving target, annotations)
```

---

## Data structures

### Core rendering

```c
/* The entire visible world */
typedef struct {
    uint32_t *pixels;   /* ARGB — one uint32_t per pixel */
    int       width, height;
} Framebuffer;

/* 2D point — used for both screen (int) and world (float) coordinates */
typedef struct { float x, y; } Vec2;

/* ARGB color packed into a single uint32_t */
typedef uint32_t Color;  /* 0xAARRGGBB */

/* Axis-aligned rectangle — used for clipping and layout */
typedef struct { int x, y, w, h; } Rect;
```

### Transform and camera

```c
/* 3×3 affine matrix — encodes translate, scale, and rotate in one multiply */
typedef struct {
    float m[3][3];
} Mat3;

/* Maps world coordinates onto the screen */
typedef struct {
    Vec2  position;      /* world-space center of the viewport */
    float zoom;          /* pixels per world unit */
    int   screen_w, screen_h;
} Camera;
```

### Math renderer

```c
/* A plottable Cartesian function y = f(x) */
typedef struct {
    float (*fn)(float x, void *ctx);
    void  *ctx;           /* parameter block for the function */
    float  x_min, x_max;
    int    samples;       /* resolution: number of x steps to evaluate */
    Color  color;
} PlotCurve;

/* A parametric curve (x(t), y(t)) — circles, spirals, gear profiles, orbits */
typedef struct {
    void  (*fn)(float t, float *x, float *y, void *ctx);
    void  *ctx;
    float  t_min, t_max;
    int    steps;
    Color  color;
} ParametricCurve;

/* A polar curve r = f(θ) — roses, cardioids, Archimedean spirals */
typedef struct {
    float (*fn)(float theta, void *ctx);
    void  *ctx;
    float  theta_min, theta_max;
    int    steps;
    Color  color;
} PolarCurve;

/* A 2D vector field F(x,y) drawn as a grid of arrows */
typedef struct {
    void  (*field)(float x, float y, float *vx, float *vy, void *ctx);
    void  *ctx;
    int    grid_cols, grid_rows;
    float  arrow_scale;
    Color  color;
} VectorField;

/* Quadratic equation ax² + bx + c = 0 (Bhaskara) */
typedef struct {
    float a, b, c;
} Quadratic;

typedef struct {
    int   root_count;   /* 0, 1, or 2 real roots */
    float r1, r2;
    float discriminant;
    float vertex_x, vertex_y;  /* apex of the parabola */
} QuadraticResult;
```

### Text rendering

```c
/* A bitmap font: each glyph stored as rows of bits */
typedef struct {
    uint8_t  glyphs[128][8];  /* 128 ASCII glyphs, 8 bytes each (8×8 grid) */
    int      glyph_w, glyph_h;
    int      spacing;         /* horizontal gap between characters */
} BitmapFont;
```

### Render pipeline

```c
/* Draw command submitted to the render queue */
typedef enum { LAYER_BG = 0, LAYER_WORLD = 1, LAYER_FX = 2, LAYER_UI = 3 } RenderLayer;

typedef struct {
    RenderLayer layer;
    int         z;            /* sort key within a layer */
    void       (*draw)(void *cmd, Framebuffer *fb, Camera *cam);
    void       *cmd;
} DrawCommand;

/* Blending modes */
typedef enum { BLEND_NONE, BLEND_ALPHA, BLEND_ADDITIVE } BlendMode;
```

### Sprite and texture atlas

```c
typedef struct {
    uint32_t *pixels;
    int       w, h;
} Texture;

typedef struct { int x, y, w, h; } AtlasRegion;

typedef struct {
    Texture     atlas;
    AtlasRegion regions[256];
    int         region_count;
} TextureAtlas;
```

### Simulation

```c
typedef struct {
    Vec2  pos;
    Vec2  vel;
    float speed;            /* scalar speed in world units/sec */
    float turn_rate;        /* maximum steering rate in radians/sec */
    Vec2  trail[512];       /* circular buffer of past positions */
    int   trail_head;
    int   trail_len;
} Missile;

typedef struct {
    Vec2  pos;
    Vec2  vel;              /* zero for a static target */
    float radius;           /* intercept detection radius */
} Target;

typedef struct {
    Missile missile;
    Target  target;
    float   time;
    int     hit;
} SimState;
```

### Event system

```c
typedef enum {
    EVENT_MISSILE_LAUNCHED,
    EVENT_TARGET_HIT,
    EVENT_SIMULATION_RESET,
    EVENT_KEY_PRESSED,
} EventType;

typedef struct {
    EventType type;
    union {
        struct { Vec2 pos; float speed; } missile;
        struct { int keycode; }          key;
    } data;
} Event;

typedef void (*EventHandler)(Event *e, void *ctx);
```

---

## Core methods

### Layer 0–1 — Pixel foundation

```c
Framebuffer fb_create(int w, int h);
void        fb_clear(Framebuffer *fb, Color bg);
void        fb_present(Framebuffer *fb);          /* blit to SDL surface */
void        draw_pixel(Framebuffer *fb, int x, int y, Color c);
                                                  /* checks bounds + clip stack + blend mode */
```

### Layer 2 — Primitives

```c
/* Lines */
void draw_line(Framebuffer *fb, int x0, int y0, int x1, int y1, Color c); /* Bresenham */

/* Outlines */
void draw_rect(Framebuffer *fb, int x, int y, int w, int h, Color c);
void draw_circle(Framebuffer *fb, int cx, int cy, int r, Color c);        /* midpoint */
void draw_triangle(Framebuffer *fb, Vec2 a, Vec2 b, Vec2 c, Color col);
void draw_polygon(Framebuffer *fb, Vec2 *pts, int n, Color c);

/* Filled */
void draw_filled_rect(Framebuffer *fb, int x, int y, int w, int h, Color c);
void draw_filled_triangle(Framebuffer *fb, Vec2 a, Vec2 b, Vec2 c, Color col); /* scanline */
void draw_filled_circle(Framebuffer *fb, int cx, int cy, int r, Color c);

/* Bézier curves */
void draw_bezier_quad(Framebuffer *fb, Vec2 p0, Vec2 p1, Vec2 p2,
                      int steps, Color c);
void draw_bezier_cubic(Framebuffer *fb, Vec2 p0, Vec2 p1, Vec2 p2, Vec2 p3,
                       int steps, Color c);
Vec2 bezier_cubic_point(Vec2 p0, Vec2 p1, Vec2 p2, Vec2 p3, float t);
```

### Layer 3 — Math renderer

```c
/* Coordinate system */
Vec2 world_to_screen(Vec2 world, Camera *cam);
Vec2 screen_to_world(Vec2 screen, Camera *cam);
void draw_axes(Framebuffer *fb, Camera *cam, Color c);
void draw_grid(Framebuffer *fb, Camera *cam, float step, Color c);
void draw_axis_ticks(Framebuffer *fb, Camera *cam, BitmapFont *font,
                     float x_step, float y_step, Color c);

/* Function plots */
void plot_curve(Framebuffer *fb, Camera *cam, PlotCurve *curve);
void plot_parametric(Framebuffer *fb, Camera *cam, ParametricCurve *curve);
void plot_polar(Framebuffer *fb, Camera *cam, PolarCurve *curve);

/* Vector field */
void draw_vector_field(Framebuffer *fb, Camera *cam, VectorField *vf);

/* Quadratic / Bhaskara */
QuadraticResult quadratic_solve(Quadratic q);
void            draw_quadratic(Framebuffer *fb, Camera *cam, Quadratic q, Color c);
void            mark_roots(Framebuffer *fb, Camera *cam, QuadraticResult r,
                            BitmapFont *font, Color c);

/* Engineering annotation */
void draw_dimension_line(Framebuffer *fb, Camera *cam, Vec2 a, Vec2 b,
                         float value, const char *unit, BitmapFont *font, Color c);
```

### Layer 4 — Transforms and camera

```c
Mat3 mat3_identity(void);
Mat3 mat3_translate(float dx, float dy);
Mat3 mat3_scale(float sx, float sy);
Mat3 mat3_rotate(float angle_rad);
Mat3 mat3_mul(Mat3 a, Mat3 b);
Vec2 mat3_apply(Mat3 m, Vec2 v);

void camera_pan(Camera *cam, Vec2 delta);
void camera_zoom(Camera *cam, float factor, Vec2 anchor);  /* zoom toward screen point */
```

### Layer 5 — Render pipeline

```c
/* Layered queue */
void render_queue_submit(DrawCommand cmd);
void render_queue_flush(Framebuffer *fb, Camera *cam);  /* sort by layer+z, execute */
void render_queue_clear(void);

/* Clip stack (scissor rect) */
void clip_push(Rect r);
void clip_pop(void);
Rect clip_current(void);

/* Blend mode */
void  blend_set(BlendMode mode);
Color blend_colors(Color src, Color dst, BlendMode mode);

/* Text */
void draw_char(Framebuffer *fb, BitmapFont *font, char c, int x, int y, Color col);
void draw_text(Framebuffer *fb, BitmapFont *font, const char *str,
               int x, int y, Color col);
void draw_textf(Framebuffer *fb, BitmapFont *font, int x, int y, Color col,
                const char *fmt, ...);   /* formatted numbers for HUD readouts */

/* Sprites and atlas */
void draw_sprite_region(Framebuffer *fb, TextureAtlas *atlas, int region_id,
                        int x, int y, Color tint);
```

### Layer 6 — Simulation

```c
/* Kinematics and guidance */
Vec2 proportional_navigation(Missile *m, Target *t, float dt);  /* PN guidance law */
void missile_update(Missile *m, Target *t, float dt);
int  check_intercept(Missile *m, Target *t);

/* Target movement */
void target_update(Target *t, float dt);

/* Simulation step */
void sim_step(SimState *s, float dt);
void sim_render(SimState *s, Framebuffer *fb, Camera *cam, BitmapFont *font);
void draw_trail(Framebuffer *fb, Camera *cam, Missile *m, Color c);

/* Predicted intercept path */
void draw_predicted_path(Framebuffer *fb, Camera *cam, SimState *s,
                         int steps, Color c);
```

### Supporting systems

```c
/* Asset manager */
Texture    *assets_load_texture(const char *path);
BitmapFont *assets_load_font(const char *path);
void        assets_unload_all(void);

/* Event bus */
void event_subscribe(EventType type, EventHandler handler, void *ctx);
void event_fire(Event *e);
void event_flush(void);    /* process all queued events once per frame */

/* Input */
void input_poll(void);
int  key_held(int keycode);
int  key_pressed(int keycode);   /* true only on the frame it was first pressed */
Vec2 mouse_world_pos(Camera *cam);

/* Fixed delta-time loop */
void engine_run(void (*update)(float dt), void (*render)(Framebuffer *fb));
```

---

## Milestone checklist

### Foundation

- [ ] Framebuffer allocated and presented via SDL
- [ ] `draw_pixel` with bounds checking
- [ ] `draw_line` — Bresenham's algorithm
- [ ] `draw_rect`, `draw_circle`, `draw_triangle`, `draw_polygon` (outlines)
- [ ] `draw_filled_rect`, `draw_filled_triangle`, `draw_filled_circle`
- [ ] `draw_bezier_quad` and `draw_bezier_cubic`

### Math renderer

- [ ] Camera with world ↔ screen transform, zoom and pan
- [ ] `draw_axes` and `draw_grid`
- [ ] `draw_axis_ticks` with numeric labels (requires bitmap font)
- [ ] `plot_curve` — generic `f(x)` plotter
- [ ] `quadratic_solve` + `draw_quadratic` + `mark_roots` (Bhaskara)
- [ ] `plot_parametric` — circle, ellipse, and spiral as first tests
- [ ] `plot_polar` — rose curve and cardioid
- [ ] `draw_vector_field` — test with a radial field
- [ ] `draw_dimension_line` — engineering annotation

### Render pipeline

- [ ] `BitmapFont` + `draw_text` + `draw_textf`
- [ ] `Mat3` transforms — translate, scale, rotate
- [ ] Layered `RenderQueue` with z-sort
- [ ] Clip stack (`clip_push` / `clip_pop`)
- [ ] `BLEND_ALPHA` and `BLEND_ADDITIVE` modes
- [ ] `TextureAtlas` + `draw_sprite_region`

### Systems

- [ ] Fixed delta-time loop with FPS counter
- [ ] Keyboard and mouse input
- [ ] Event bus (`event_subscribe` / `event_fire` / `event_flush`)
- [ ] Asset manager — load/cache textures and fonts

### Simulation

- [ ] Static missile to static target — proportional navigation
- [ ] Moving target intercept
- [ ] Trajectory trail with additive blending
- [ ] Predicted intercept path overlay
- [ ] HUD overlay: speed, distance-to-target, time elapsed, angle-of-attack

---

## Recommended next steps

Once the milestones above are complete:

**Math extensions** — implicit curve renderer (`f(x,y) = 0` as isolines), numerical integration display (Riemann sums, trapezoid), differential equation field solver and phase portrait plotter, Fourier series visualizer.

**Simulation extensions** — multi-body simulation (N missiles, N targets), projectile drag and wind models, 2D rigid body physics (moment of inertia, torque), spring-mass systems for structural simulation.

**Renderer extensions** — offscreen render targets (render to texture), stencil masking, subpixel rendering for smoother curves at low resolution, sprite animation (frame cycling over time).

**Engineering tools** — graph overlay system (plot live telemetry over the simulation view), CSV/data file loader for plotting measured data alongside analytical curves, export current view to PNG or SVG.

---

## Mental model

```
draw_pixel()              ← atomic unit
      ↓
Bresenham + scanline      ← lines, shapes, filled geometry
      ↓
Bézier + parametric       ← smooth curves, paths
      ↓
plot_curve / axes         ← mathematics made visible
      ↓
world ↔ screen / Mat3     ← camera ties it all together
      ↓
blend + clip + text       ← pipeline gives it structure
      ↓
missile + guidance law    ← physics running in world space
      ↓
HUD + annotations         ← the numbers that explain the picture
```

---

*Built from scratch. No magic. Just pixels — and the mathematics that place them.*