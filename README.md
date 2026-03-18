# Oytan 2D Engine

## Why the name?

**Oytan** fuses two Tupi words — *oby* (green and blue) and *pytã* (red) — the indigenous language of Brazil's first peoples, and the origin of our fundamental RGB triad. Every pixel this engine draws traces back to that root.

---

## What Oytan is

Oytan is a **C software renderer** built from the ground up — no GPU pipeline, no external rendering libs. It starts with a flat array of pixels in memory and builds upward through geometry, math visualization, and finally physical simulation of a teleguided missile.

The goal isn't just a game engine. It's a **visible mathematics machine**: draw a parabola, watch Bhaskara's roots appear as intersections, then launch a missile that follows that exact trajectory toward a moving target.

---

## Architecture

Each layer depends strictly on the one below. You cannot skip.

| Layer | Name | Purpose |
|---|---|---|
| 0 | Framebuffer | Raw pixel memory, clear/present loop |
| 1 | `drawPixel` | Atomic write with bounds check |
| 2 | Primitives | Lines, shapes, filled geometry |
| 3 | Math renderer | Functions, curves, axes, Bhaskara |
| 4 | Transforms + Camera | World↔screen mapping, zoom/pan |
| 5 | Simulation | Missile, kinematics, moving target |
| 6 | Game loop + UI | Fixed `dt`, input, HUD, scene manager |

---

## Data structures

### Core rendering

```c
/* The entire visible world */
typedef struct {
    uint32_t *pixels;   /* ARGB — one uint32_t per pixel */
    int width, height;
} Framebuffer;

/* A point in 2D space (integer screen coords OR float world coords) */
typedef struct { float x, y; } Vec2;

/* ARGB color packed into a uint32_t */
typedef uint32_t Color;  /* 0xAARRGGBB */
```

### Transform and camera

```c
/* 3×3 affine matrix — handles translate, scale, rotate in one multiply */
typedef struct {
    float m[3][3];
} Mat3;

typedef struct {
    Vec2  position;   /* world-space center */
    float zoom;       /* pixels per world unit */
    int   screen_w, screen_h;
} Camera;
```

### Math renderer

```c
/* A plottable function f(x) → y, with domain and color */
typedef struct {
    float (*fn)(float x, void *ctx);  /* function pointer + context for params */
    void  *ctx;
    float  x_min, x_max;
    Color  color;
    int    samples;                   /* resolution: how many x steps to evaluate */
} PlotCurve;

/* Bhaskara / quadratic formula — ax² + bx + c = 0 */
typedef struct {
    float a, b, c;
} Quadratic;

typedef struct {
    int   has_roots;     /* 0 = no real roots, 1 = one, 2 = two */
    float r1, r2;        /* real roots (x-intercepts) */
    float discriminant;
} QuadraticResult;
```

### Missile simulation

```c
typedef struct {
    Vec2  pos;         /* world position */
    Vec2  vel;         /* velocity vector */
    float speed;       /* scalar speed (m/s in world units) */
    float turn_rate;   /* max radians/sec the missile can steer */
    float trail[512];  /* circular buffer of past positions for trail drawing */
    int   trail_head;
} Missile;

typedef struct {
    Vec2  pos;
    Vec2  vel;         /* zero for static target, non-zero for moving */
    float radius;      /* hit detection radius */
} Target;

typedef struct {
    Missile missile;
    Target  target;
    float   time;
    int     hit;       /* 1 = intercept achieved */
} SimState;
```

### Scene and entity management

```c
typedef struct Entity {
    Vec2  pos;
    Vec2  vel;
    void  (*update)(struct Entity *self, float dt);
    void  (*render)(struct Entity *self, Framebuffer *fb, Camera *cam);
    void  *data;   /* payload — cast to Missile*, Target*, etc. */
} Entity;

typedef struct {
    Entity *entities;
    int     count, capacity;
} Scene;
```

---

## Core methods by layer

### Layer 0–1 — Pixel foundation

```c
Framebuffer fb_create(int w, int h);
void        fb_clear(Framebuffer *fb, Color bg);
void        fb_present(Framebuffer *fb);           /* blit to SDL surface / OpenGL tex */
void        draw_pixel(Framebuffer *fb, int x, int y, Color c);
```

### Layer 2 — Primitives

```c
/* Lines */
void draw_line(Framebuffer *fb, int x0, int y0, int x1, int y1, Color c);   /* Bresenham */

/* Outlines */
void draw_rect(Framebuffer *fb, int x, int y, int w, int h, Color c);
void draw_circle(Framebuffer *fb, int cx, int cy, int r, Color c);           /* midpoint */
void draw_triangle(Framebuffer *fb, Vec2 a, Vec2 b, Vec2 c, Color col);
void draw_polygon(Framebuffer *fb, Vec2 *pts, int n, Color c);

/* Filled */
void draw_filled_rect(Framebuffer *fb, int x, int y, int w, int h, Color c);
void draw_filled_triangle(Framebuffer *fb, Vec2 a, Vec2 b, Vec2 c, Color col); /* scanline */
void draw_filled_circle(Framebuffer *fb, int cx, int cy, int r, Color c);
```

### Layer 3 — Math renderer

```c
/* Coordinate system */
void        draw_axes(Framebuffer *fb, Camera *cam, Color c);
Vec2        world_to_screen(Vec2 world, Camera *cam);
Vec2        screen_to_world(Vec2 screen, Camera *cam);

/* Function plotting */
void        plot_curve(Framebuffer *fb, Camera *cam, PlotCurve *curve);

/* Quadratic / Bhaskara */
QuadraticResult quadratic_solve(Quadratic q);
void            draw_quadratic(Framebuffer *fb, Camera *cam, Quadratic q, Color c);
void            mark_roots(Framebuffer *fb, Camera *cam, QuadraticResult r, Color c);
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
void camera_zoom(Camera *cam, float factor, Vec2 anchor);   /* zoom toward anchor */
```

### Layer 5 — Missile simulation

```c
/* Guidance */
Vec2  proportional_navigation(Missile *m, Target *t, float dt); /* PN guidance law */
void  missile_update(Missile *m, Target *t, float dt);
int   check_intercept(Missile *m, Target *t);

/* Target */
void  target_update(Target *t, float dt);   /* moves target along its velocity */

/* Simulation step */
void  sim_step(SimState *s, float dt);
void  sim_render(SimState *s, Framebuffer *fb, Camera *cam);
void  draw_trail(Framebuffer *fb, Camera *cam, Missile *m, Color c);
```

### Layer 6 — Loop and input

```c
void input_poll(void);
int  key_held(int keycode);
Vec2 mouse_world_pos(Camera *cam);

void game_loop_run(Scene *scene, Framebuffer *fb, Camera *cam);
void hud_draw(Framebuffer *fb, SimState *s);
```

---

## Milestone checklist

- [ ] Framebuffer allocated and presented via SDL
- [ ] `draw_pixel` with bounds checking
- [ ] `draw_line` — Bresenham's algorithm
- [ ] `draw_rect`, `draw_circle`, `draw_triangle`, `draw_polygon` (outlines)
- [ ] `draw_filled_rect`, `draw_filled_triangle`, `draw_filled_circle`
- [ ] Camera with world↔screen transform, zoom and pan
- [ ] `draw_axes` — grid with labeled ticks in world space
- [ ] `plot_curve` — generic `f(x)` plotter with configurable domain and samples
- [ ] `quadratic_solve` + `draw_quadratic` + root markers (Bhaskara)
- [ ] `Transform2D` / `Mat3` — translation, scale, rotation
- [ ] Fixed delta-time game loop with FPS counter
- [ ] Keyboard + mouse input
- [ ] Static missile → static target intercept
- [ ] Moving target with proportional navigation guidance
- [ ] Trajectory trail renderer
- [ ] Basic `Entity` + `Scene` system
- [ ] HUD overlay (speed, distance-to-target, time)

---

## Recommended next steps

Once the milestones are complete: **spritesheet animation** (cycle frame indices over time), **collision broadphase** (spatial hash or quadtree for multiple targets), **audio** (SDL_mixer for launch/impact sounds), **asset manager** (centralized load/cache for textures), and **scenario scripting** (define launch conditions and moving target paths as data files).

---

## Mental model

```
draw_pixel()              ← atomic unit
      ↓
draw_line() / shapes      ← Bresenham + scanline fill
      ↓
plot_curve() / axes       ← math visible on screen
      ↓
world ↔ screen transform  ← camera ties it together
      ↓
missile + guidance law    ← physics running in world space
      ↓
game loop + HUD           ← it's alive
```

---

*Built from scratch. No magic. Just pixels — and one missile that knows where it's going.*