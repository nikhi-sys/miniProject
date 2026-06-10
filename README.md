/*
 * 2D Graphics Editor in C
 * Uses a 2D character canvas with '*' for shapes and '_' for background
 * Supports: Circle, Rectangle, Line, Triangle
 * Operations: Add, Delete, Modify objects
 */

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <string.h>

/* ──────────────────────── Canvas Settings ──────────────────────── */
#define ROWS      30
#define COLS      60
#define BG        '_'
#define DRAW_CH   '*'
#define MAX_OBJ   50

/* ──────────────────────── Shape Types ──────────────────────── */
typedef enum {
    SHAPE_CIRCLE = 1,
    SHAPE_RECT,
    SHAPE_LINE,
    SHAPE_TRIANGLE
} ShapeType;

/* ──────────────────────── Object Struct ──────────────────────── */
typedef struct {
    int       id;
    ShapeType type;
    int       x1, y1;   /* anchor / center / start point */
    int       x2, y2;   /* second point (end / corner)   */
    int       x3, y3;   /* third vertex (triangle only)  */
    int       r;         /* radius (circle only)          */
    int       active;    /* 1 = present, 0 = deleted      */
} Object;

/* ──────────────────────── Globals ──────────────────────── */
char    canvas[ROWS][COLS];
Object  objects[MAX_OBJ];
int     obj_count = 0;
int     next_id   = 1;

/* ═══════════════════════════════════════════════════════════════
 *  CANVAS HELPERS
 * ═══════════════════════════════════════════════════════════════ */

void clear_canvas(void) {
    for (int r = 0; r < ROWS; r++)
        for (int c = 0; c < COLS; c++)
            canvas[r][c] = BG;
}

void set_pixel(int r, int c) {
    if (r >= 0 && r < ROWS && c >= 0 && c < COLS)
        canvas[r][c] = DRAW_CH;
}

void display_canvas(void) {
    printf("\n");
    /* Column ruler */
    printf("   ");
    for (int c = 0; c < COLS; c++) printf("%d", c % 10);
    printf("\n   +");
    for (int c = 0; c < COLS; c++) printf("-");
    printf("+\n");

    for (int r = 0; r < ROWS; r++) {
        printf("%2d |", r);
        for (int c = 0; c < COLS; c++)
            putchar(canvas[r][c]);
        printf("|\n");
    }

    printf("   +");
    for (int c = 0; c < COLS; c++) printf("-");
    printf("+\n\n");
}

/* ═══════════════════════════════════════════════════════════════
 *  DRAW PRIMITIVES  (draw a single object onto the canvas)
 * ═══════════════════════════════════════════════════════════════ */

/* Bresenham line */
void draw_line(int r1, int c1, int r2, int c2) {
    int dr = abs(r2 - r1), dc = abs(c2 - c1);
    int sr = (r1 < r2) ? 1 : -1;
    int sc = (c1 < c2) ? 1 : -1;
    int err = dr - dc;

    for (;;) {
        set_pixel(r1, c1);
        if (r1 == r2 && c1 == c2) break;
        int e2 = 2 * err;
        if (e2 > -dc) { err -= dc; r1 += sr; }
        if (e2 <  dr) { err += dr; c1 += sc; }
    }
}

/* Midpoint circle */
void draw_circle(int cr, int cc, int radius) {
    int x = 0, y = radius;
    int d = 1 - radius;

    while (x <= y) {
        set_pixel(cr + x, cc + y);  set_pixel(cr - x, cc + y);
        set_pixel(cr + x, cc - y);  set_pixel(cr - x, cc - y);
        set_pixel(cr + y, cc + x);  set_pixel(cr - y, cc + x);
        set_pixel(cr + y, cc - x);  set_pixel(cr - y, cc - x);

        if (d < 0)       d += 2 * x + 3;
        else           { d += 2 * (x - y) + 5; y--; }
        x++;
    }
}

/* Axis-aligned rectangle (outline) */
void draw_rect(int r1, int c1, int r2, int c2) {
    for (int c = c1; c <= c2; c++) { set_pixel(r1, c); set_pixel(r2, c); }
    for (int r = r1; r <= r2; r++) { set_pixel(r, c1); set_pixel(r, c2); }
}

/* Triangle: three vertices */
void draw_triangle(int r1, int c1, int r2, int c2, int r3, int c3) {
    draw_line(r1, c1, r2, c2);
    draw_line(r2, c2, r3, c3);
    draw_line(r3, c3, r1, c1);
}

/* ═══════════════════════════════════════════════════════════════
 *  FULL REDRAW  (wipe canvas and repaint every active object)
 * ═══════════════════════════════════════════════════════════════ */

void redraw_all(void) {
    clear_canvas();
    for (int i = 0; i < obj_count; i++) {
        if (!objects[i].active) continue;
        Object *o = &objects[i];
        switch (o->type) {
            case SHAPE_CIRCLE:
                draw_circle(o->y1, o->x1, o->r); break;
            case SHAPE_RECT:
                draw_rect(o->y1, o->x1, o->y2, o->x2); break;
            case SHAPE_LINE:
                draw_line(o->y1, o->x1, o->y2, o->x2); break;
            case SHAPE_TRIANGLE:
                draw_triangle(o->y1, o->x1, o->y2, o->x2, o->y3, o->x3); break;
        }
    }
}

/* ═══════════════════════════════════════════════════════════════
 *  ADD OBJECTS
 * ═══════════════════════════════════════════════════════════════ */

int add_circle(int cx, int cy, int radius) {
    if (obj_count >= MAX_OBJ) { printf("Max objects reached!\n"); return -1; }
    Object *o = &objects[obj_count++];
    o->id = next_id++; o->type = SHAPE_CIRCLE;
    o->x1 = cx; o->y1 = cy; o->r = radius; o->active = 1;
    redraw_all();
    printf("Circle added  (id=%d)\n", o->id);
    return o->id;
}

int add_rect(int c1, int r1, int c2, int r2) {
    if (obj_count >= MAX_OBJ) { printf("Max objects reached!\n"); return -1; }
    Object *o = &objects[obj_count++];
    o->id = next_id++; o->type = SHAPE_RECT;
    o->x1 = c1; o->y1 = r1; o->x2 = c2; o->y2 = r2; o->active = 1;
    redraw_all();
    printf("Rectangle added (id=%d)\n", o->id);
    return o->id;
}

int add_line(int c1, int r1, int c2, int r2) {
    if (obj_count >= MAX_OBJ) { printf("Max objects reached!\n"); return -1; }
    Object *o = &objects[obj_count++];
    o->id = next_id++; o->type = SHAPE_LINE;
    o->x1 = c1; o->y1 = r1; o->x2 = c2; o->y2 = r2; o->active = 1;
    redraw_all();
    printf("Line added    (id=%d)\n", o->id);
    return o->id;
}

int add_triangle(int c1, int r1, int c2, int r2, int c3, int r3) {
    if (obj_count >= MAX_OBJ) { printf("Max objects reached!\n"); return -1; }
    Object *o = &objects[obj_count++];
    o->id = next_id++; o->type = SHAPE_TRIANGLE;
    o->x1 = c1; o->y1 = r1; o->x2 = c2; o->y2 = r2;
    o->x3 = c3; o->y3 = r3; o->active = 1;
    redraw_all();
    printf("Triangle added (id=%d)\n", o->id);
    return o->id;
}

/* ═══════════════════════════════════════════════════════════════
 *  DELETE OBJECT
 * ═══════════════════════════════════════════════════════════════ */

int delete_object(int id) {
    for (int i = 0; i < obj_count; i++) {
        if (objects[i].id == id && objects[i].active) {
            objects[i].active = 0;
            redraw_all();
            printf("Object %d deleted.\n", id);
            return 1;
        }
    }
    printf("Object %d not found.\n", id);
    return 0;
}

/* ═══════════════════════════════════════════════════════════════
 *  MODIFY OBJECT
 *  Pass new parameters; for fields you don't want to change,
 *  pass the existing value (or use the helper wrappers below).
 * ═══════════════════════════════════════════════════════════════ */

int modify_object(int id, int x1, int y1, int x2, int y2,
                  int x3, int y3, int r)
{
    for (int i = 0; i < obj_count; i++) {
        if (objects[i].id == id && objects[i].active) {
            Object *o = &objects[i];
            o->x1 = x1; o->y1 = y1;
            o->x2 = x2; o->y2 = y2;
            o->x3 = x3; o->y3 = y3;
            o->r  = r;
            redraw_all();
            printf("Object %d modified.\n", id);
            return 1;
        }
    }
    printf("Object %d not found.\n", id);
    return 0;
}

/* ═══════════════════════════════════════════════════════════════
 *  LIST OBJECTS
 * ═══════════════════════════════════════════════════════════════ */

void list_objects(void) {
    static const char *names[] = {"", "Circle", "Rectangle", "Line", "Triangle"};
    printf("\n%-4s %-10s  Details\n", "ID", "Shape");
    printf("-------------------------------------------\n");
    int any = 0;
    for (int i = 0; i < obj_count; i++) {
        if (!objects[i].active) continue;
        any = 1;
        Object *o = &objects[i];
        printf("%-4d %-10s  ", o->id, names[o->type]);
        switch (o->type) {
            case SHAPE_CIRCLE:
                printf("center=(%d,%d) r=%d\n", o->x1, o->y1, o->r); break;
            case SHAPE_RECT:
                printf("(%d,%d)–(%d,%d)\n", o->x1, o->y1, o->x2, o->y2); break;
            case SHAPE_LINE:
                printf("(%d,%d)→(%d,%d)\n", o->x1, o->y1, o->x2, o->y2); break;
            case SHAPE_TRIANGLE:
                printf("(%d,%d) (%d,%d) (%d,%d)\n",
                       o->x1, o->y1, o->x2, o->y2, o->x3, o->y3); break;
        }
    }
    if (!any) printf("  (no objects)\n");
    printf("\n");
}

/* ═══════════════════════════════════════════════════════════════
 *  INTERACTIVE MENU
 * ═══════════════════════════════════════════════════════════════ */

static int read_int(const char *prompt) {
    int v;
    printf("%s", prompt);
    scanf("%d", &v);
    return v;
}

void menu_add(void) {
    printf("\n  1) Circle   2) Rectangle   3) Line   4) Triangle\n");
    int choice = read_int("  Shape: ");
    switch (choice) {
        case 1: {
            int cx = read_int("  Center col: ");
            int cy = read_int("  Center row: ");
            int r  = read_int("  Radius    : ");
            add_circle(cx, cy, r);
            break;
        }
        case 2: {
            int c1 = read_int("  Top-left  col: ");
            int r1 = read_int("  Top-left  row: ");
            int c2 = read_int("  Bot-right col: ");
            int r2 = read_int("  Bot-right row: ");
            add_rect(c1, r1, c2, r2);
            break;
        }
        case 3: {
            int c1 = read_int("  Start col: ");
            int r1 = read_int("  Start row: ");
            int c2 = read_int("  End   col: ");
            int r2 = read_int("  End   row: ");
            add_line(c1, r1, c2, r2);
            break;
        }
        case 4: {
            int c1 = read_int("  V1 col: "); int r1 = read_int("  V1 row: ");
            int c2 = read_int("  V2 col: "); int r2 = read_int("  V2 row: ");
            int c3 = read_int("  V3 col: "); int r3 = read_int("  V3 row: ");
            add_triangle(c1, r1, c2, r2, c3, r3);
            break;
        }
        default: printf("  Unknown shape.\n");
    }
}

void menu_modify(void) {
    list_objects();
    int id = read_int("  ID to modify: ");
    /* find the object first to get its type */
    Object *found = NULL;
    for (int i = 0; i < obj_count; i++)
        if (objects[i].id == id && objects[i].active) { found = &objects[i]; break; }
    if (!found) { printf("  Object %d not found.\n", id); return; }

    int x1=found->x1, y1=found->y1, x2=found->x2, y2=found->y2,
        x3=found->x3, y3=found->y3, r=found->r;

    switch (found->type) {
        case SHAPE_CIRCLE:
            x1 = read_int("  New center col: ");
            y1 = read_int("  New center row: ");
            r  = read_int("  New radius    : ");
            break;
        case SHAPE_RECT:
            x1 = read_int("  New top-left  col: ");
            y1 = read_int("  New top-left  row: ");
            x2 = read_int("  New bot-right col: ");
            y2 = read_int("  New bot-right row: ");
            break;
        case SHAPE_LINE:
            x1 = read_int("  New start col: ");
            y1 = read_int("  New start row: ");
            x2 = read_int("  New end   col: ");
            y2 = read_int("  New end   row: ");
            break;
        case SHAPE_TRIANGLE:
            x1 = read_int("  New V1 col: "); y1 = read_int("  New V1 row: ");
            x2 = read_int("  New V2 col: "); y2 = read_int("  New V2 row: ");
            x3 = read_int("  New V3 col: "); y3 = read_int("  New V3 row: ");
            break;
    }
    modify_object(id, x1, y1, x2, y2, x3, y3, r);
}

void run_demo(void) {
    printf("=== Demo: adding shapes ===\n");
    add_circle(20, 14, 8);
    add_rect(35, 5, 55, 24);
    add_line(1, 1, 58, 28);
    add_triangle(10, 2, 30, 2, 20, 14);
    display_canvas();

    printf("=== Demo: deleting the line (id=3) ===\n");
    delete_object(3);
    display_canvas();

    printf("=== Demo: moving the circle (id=1) to col=40, row=14 ===\n");
    modify_object(1, 40, 14, 0, 0, 0, 0, 8);
    display_canvas();
    list_objects();
}

/* ═══════════════════════════════════════════════════════════════
 *  MAIN
 * ═══════════════════════════════════════════════════════════════ */

int main(void) {
    clear_canvas();

    printf("╔══════════════════════════════════╗\n");
    printf("║   2D ASCII Graphics Editor (C)   ║\n");
    printf("║   shapes: * | background: _      ║\n");
    printf("╚══════════════════════════════════╝\n");
    printf("\n  1) Interactive mode\n  2) Run demo\n");
    int mode = read_int("  Choose: ");

    if (mode == 2) { run_demo(); return 0; }

    /* ── Interactive loop ── */
    for (;;) {
        printf("\n┌─ Menu ───────────────────────────┐\n");
        printf("│ 1) Add shape                     │\n");
        printf("│ 2) Delete shape                  │\n");
        printf("│ 3) Modify shape                  │\n");
        printf("│ 4) Display canvas                │\n");
        printf("│ 5) List objects                  │\n");
        printf("│ 0) Quit                          │\n");
        printf("└──────────────────────────────────┘\n");

        int choice = read_int("  > ");
        switch (choice) {
            case 1: menu_add();    display_canvas(); break;
            case 2: {
                list_objects();
                int id = read_int("  ID to delete: ");
                delete_object(id);
                display_canvas();
                break;
            }
            case 3: menu_modify(); display_canvas(); break;
            case 4: display_canvas(); break;
            case 5: list_objects();   break;
            case 0: printf("Bye!\n"); return 0;
            default: printf("  Unknown option.\n");
        }
    }
}
