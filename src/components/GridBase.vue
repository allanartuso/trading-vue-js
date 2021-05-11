<template>
  <div :class="gridClass" :style="style">
    <canvas
      ref="canvas"
      :style="style"
      @mousemove="mousemove"
      @mouseout="mouseout"
      @mouseup="mouseup"
      @mousedown="mousedown"
    />
  </div>
</template>

<script>
// Grid.js listens to various user-generated events,
// emits Vue-events if something has changed (e.g. range)
// Think of it as an I/O system for Grid.vue

import FrameAnimation from "../stuff/frame.js";
import * as Hammer from "hammerjs";
import Hamster from "hamsterjs";
import Utils from "../stuff/utils.js";
import math from "../stuff/math.js";

// Grid is good.
export default {
  name: "GridBase",
  props: [
    "gridClass",
    "cursor",
    "layoutGrids",
    "grid_id",
    "propRange",
    "sub",
    "interval",
    "shaders",
    "colors",

    "meta",
    "y_transform",
    "config",
    "tv_id",
    "font",
    "attrs",
    "position",

    "SCROLL_WHEEL",
    "MIN_ZOOM",
    "MAX_ZOOM",
    "ZOOM_MODE",
  ],
  data() {
    return {
      canvas: undefined,
      ctx: undefined,
      id: undefined,
      layout: undefined,
      wmode: undefined,
      range: undefined,
      offset_x: 0,
      offset_y: 0,
      deltas: 0, // Wheel delta events
      overlays: [],
      style: {
        left: this.position.x + "px",
        top: this.position.y + "px",
        position: "absolute",
      },
    };
  },
  computed: {},
  created() {
    this.id = this.grid_id;
    this.layout = this.layoutGrids[this.id];
    this.wmode = this.SCROLL_WHEEL;
    this.range = this.propRange;
  },

  mounted() {
    this.canvas = this.$refs.canvas;
    this.ctx = this.canvas.getContext("2d");
    this.$nextTick(() => {
      this.setup();
      this.listeners();
    });
  },

  beforeDestroy() {
    let rm = removeEventListener;
    rm("gesturestart", this.gesturestart);
    rm("gesturechange", this.gesturechange);
    rm("gestureend", this.gestureend);
    if (this.mc) this.mc.destroy();
    if (this.hm) this.hm.unwheel();
  },
  methods: {
    setup() {
      const layout = this.layout;

      let dpr = window.devicePixelRatio || 1;
      this.canvas.style.width = `${this.attrs.width}px`;
      this.canvas.style.height = `${this.attrs.height}px`;

      // TODO: fix it, set directcly in the element? Look canvas.js create_canvas. Add elements that are inside hs, add overlays
      this.canvas.attrs = {
        id: `${this.tv_id}-${this.grid_id}-canvas`,
        width: layout.width,
        height: layout.height,
        overflow: "hidden",
      };

      if (dpr < 1) dpr = 1; // Realy ? That's it? Issue #63
      this.$nextTick(() => {
        var rect = this.canvas.getBoundingClientRect();
        this.canvas.width = rect.width * dpr;
        this.canvas.height = rect.height * dpr;
        const ctx = this.canvas.getContext("2d", {
          // TODO: test the boost:
          //alpha: false,
          //desynchronized: true,
          //preserveDrawingBuffer: false
        });
        ctx.scale(dpr, dpr);
        this.update();
        // Fallback fix for Brave browser
        // https://github.com/brave/brave-browser/issues/1738
        if (!ctx.measureTextOrg) {
          ctx.measureTextOrg = ctx.measureText;
        }
        ctx.measureText = (text) => Utils.measureText(ctx, text, this.tv_id);
      });
    },
    listeners() {
      this.hm = Hamster(this.canvas);
      this.hm.wheel((event, delta) => this.mousezoom(-delta * 50, event));

      let mc = (this.mc = new Hammer.Manager(this.canvas));
      let T = Utils.is_mobile ? 10 : 0;
      mc.add(new Hammer.Pan({ threshold: T }));
      mc.add(new Hammer.Tap());
      mc.add(new Hammer.Pinch({ threshold: 0 }));
      mc.get("pinch").set({ enable: true });
      if (Utils.is_mobile) mc.add(new Hammer.Press());

      mc.on("panstart", (event) => {
        if (this.cursor.scroll_lock) return;
        if (this.cursor.mode === "aim") {
          return this.emit_cursor_coord(event);
        }
        let tfrm = this.y_transform;
        this.drug = {
          x: event.center.x + this.offset_x,
          y: event.center.y + this.offset_y,
          r: this.range.slice(),
          t: this.range[1] - this.range[0],
          o: tfrm ? tfrm.offset || 0 : 0,
          y_r: tfrm && tfrm.range ? tfrm.range.slice() : undefined,
          B: this.layout.B,
          t0: Utils.now(),
        };
        this.$emit("cursor-changed", {
          grid_id: this.id,
          x: event.center.x + this.offset_x,
          y: event.center.y + this.offset_y,
        });
        this.$emit("cursor-locked", true);
      });

      mc.on("panmove", (event) => {
        if (Utils.is_mobile) {
          this.calc_offset();
          this.propagate("mousemove", this.touch2mouse(event));
        }
        if (this.drug) {
          this.mousedrag(
            this.drug.x + event.deltaX,
            this.drug.y + event.deltaY
          );
          this.$emit("cursor-changed", {
            grid_id: this.id,
            x: event.center.x + this.offset_x,
            y: event.center.y + this.offset_y,
          });
        } else if (this.cursor.mode === "aim") {
          this.emit_cursor_coord(event);
        }
      });

      mc.on("panend", (event) => {
        if (Utils.is_mobile && this.drug) {
          this.pan_fade(event);
        }
        this.drug = null;
        this.$emit("cursor-locked", false);
      });

      mc.on("tap", (event) => {
        if (!Utils.is_mobile) return;
        this.sim_mousedown(event);
        if (this.fade) this.fade.stop();
        this.$emit("cursor-changed", {});
        this.$emit("cursor-changed", {
          /*grid_id: this.id,
                x: undefined,//event.center.x + this.offset_x,
                y: undefined,//event.center.y + this.offset_y,*/
          mode: "explore",
        });
        this.update();
      });

      mc.on("pinchstart", () => {
        this.drug = null;
        this.pinch = {
          t: this.range[1] - this.range[0],
          r: this.range.slice(),
        };
      });

      mc.on("pinchend", () => {
        this.pinch = null;
      });

      mc.on("pinch", (event) => {
        if (this.pinch) this.pinchzoom(event.scale);
      });

      mc.on("press", (event) => {
        if (!Utils.is_mobile) return;
        if (this.fade) this.fade.stop();
        this.calc_offset();
        this.emit_cursor_coord(event, { mode: "aim" });
        setTimeout(() => this.update());
        this.sim_mousedown(event);
      });

      let add = addEventListener;
      add("gesturestart", this.gesturestart);
      add("gesturechange", this.gesturechange);
      add("gestureend", this.gestureend);
    },

    gesturestart(event) {
      event.preventDefault();
    },
    gesturechange(event) {
      event.preventDefault();
    },
    gestureend(event) {
      event.preventDefault();
    },

    mousemove(event) {
      if (Utils.is_mobile) return;
      this.$emit("cursor-changed", {
        grid_id: this.id,
        x: event.layerX,
        y: event.layerY + this.layout.offset,
      });
      this.calc_offset();
      this.propagate("mousemove", event);
    },

    mouseout(event) {
      if (Utils.is_mobile) return;
      this.$emit("cursor-changed", {});
      this.propagate("mouseout", event);
    },

    mouseup(event) {
      this.drug = null;
      this.$emit("cursor-locked", false);
      this.propagate("mouseup", event);
    },
    mousedown(event) {
      if (Utils.is_mobile) return;
      this.propagate("mousedown", event);
      this.$emit("cursor-locked", true);
      if (event.defaultPrevented) return;
      this.$emit("custom-event", {
        event: "grid-mousedown",
        args: [this.id, event],
      });
    },
    // Simulated mousedown (for mobile)
    sim_mousedown(event) {
      if (event.srcEvent.defaultPrevented) return;
      this.$emit("custom-event", {
        event: "grid-mousedown",
        args: [this.id, event],
      });
      this.propagate("mousemove", this.touch2mouse(event));
      this.update();
      this.propagate("mousedown", this.touch2mouse(event));
      setTimeout(() => {
        this.propagate("click", this.touch2mouse(event));
      });
    },
    // Convert touch to "mouse" event
    touch2mouse(e) {
      this.calc_offset();
      return {
        original: e.srcEvent,
        layerX: e.center.x + this.offset_x,
        layerY: e.center.y + this.offset_y,
        preventDefault: function() {
          this.original.preventDefault();
        },
      };
    },

    click(event) {
      this.propagate("click", event);
    },
    emit_cursor_coord(event, add = {}) {
      this.$emit(
        "cursor-changed",
        Object.assign(
          {
            grid_id: this.id,
            x: event.center.x + this.offset_x,
            y: event.center.y + this.offset_y + this.layout.offset,
          },
          add
        )
      );
    },

    pan_fade(event) {
      let dt = Utils.now() - this.drug.t0;
      let dx = this.range[1] - this.drug.r[1];
      let v = (42 * dx) / dt;
      let v0 = Math.abs(v * 0.01);
      if (dt > 500) return;
      if (this.fade) this.fade.stop();
      this.fade = new FrameAnimation((self) => {
        v *= 0.85;
        if (Math.abs(v) < v0) {
          self.stop();
        }
        this.range[0] += v;
        this.range[1] += v;
        this.change_range();
      });
    },

    calc_offset() {
      let rect = this.canvas.getBoundingClientRect();
      this.offset_x = -rect.x;
      this.offset_y = -rect.y;
    },

    new_layer(layer) {
      if (layer.name === "crosshair") {
        this.crosshair = layer;
      } else {
        this.overlays.push(layer);
      }
      this.update();
    },

    del_layer(id) {
      this.overlays = this.overlays.filter((x) => x.id !== id);
      this.update();
    },

    show_hide_layer(event) {
      let l = this.overlays.filter((x) => x.id === event.id);
      if (l.length) l[0].display = event.display;
    },

    update() {
      // Update reference to the grid
      // TODO: check what happens if data changes interval

      //   this.layout = this.layoutGrids[this.id];
      //   this.interval = this.interval;

      if (!this.layout) return;

      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      if (this.shaders.length) this.apply_shaders();

      this.grid();

      let overlays = [];
      overlays.push(...this.overlays);

      // z-index sorting
      overlays.sort((l1, l2) => l1.z - l2.z);

      overlays.forEach((l) => {
        if (!l.display) return;
        this.ctx.save();
        let r = l.renderer;
        if (r.pre_draw) r.pre_draw(this.ctx);
        r.draw(this.ctx);
        if (r.post_draw) r.post_draw(this.ctx);
        this.ctx.restore();
      });

      if (this.crosshair) {
        this.crosshair.renderer.draw(this.ctx);
      }
    },
    apply_shaders() {
      let layout = this.layoutGrids[this.id];
      let props = {
        layout: layout,
        range: this.range,
        interval: this.interval,
        tf: layout.ti_map.tf,
        cursor: this.cursor,
        colors: this.colors,
        sub: this.sub,
        font: this.font,
        config: this.config,
        meta: this.meta,
      };
      for (var s of this.shaders) {
        this.ctx.save();
        s.draw(this.ctx, props);
        this.ctx.restore();
      }
    },
    // Actually draws the grid (for real)
    grid() {
      this.ctx.strokeStyle = this.colors.grid;
      this.ctx.beginPath();

      const ymax = this.layout.height;
      for (var [x, p] of this.layout.xs) {
        this.ctx.moveTo(x - 0.5, 0);
        this.ctx.lineTo(x - 0.5, ymax);
      }

      for (var [y, y$] of this.layout.ys) {
        this.ctx.moveTo(0, y - 0.5);
        this.ctx.lineTo(this.layout.width, y - 0.5);
      }

      this.ctx.stroke();

      if (this.grid_id) this.upper_border();
    },
    upper_border() {
      this.ctx.strokeStyle = this.colors.scale;
      this.ctx.beginPath();
      this.ctx.moveTo(0, 0.5);
      this.ctx.lineTo(this.layout.width, 0.5);
      this.ctx.stroke();
    },
    mousezoom(delta, event) {
      // TODO: for mobile
      if (this.wmode !== "pass") {
        if (this.wmode === "click" && !this.meta.activated) {
          return;
        }
        event.originalEvent.preventDefault();
        event.preventDefault();
      }

      event.deltaX = event.deltaX || Utils.get_deltaX(event);
      event.deltaY = event.deltaY || Utils.get_deltaY(event);

      if (Math.abs(event.deltaX) > 0) {
        this.trackpad = true;
        if (Math.abs(event.deltaX) >= Math.abs(event.deltaY)) {
          delta *= 0.1;
        }
        this.trackpad_scroll(event);
      }

      if (this.trackpad) delta *= 0.032;

      delta = Utils.smart_wheel(delta);

      // TODO: mouse zooming is a little jerky,
      // needs to follow f(mouse_wheel_speed) and
      // if speed is low, scroll shoud be slower
      if (delta < 0 && this.sub.length <= this.MIN_ZOOM) return;
      if (delta > 0 && this.sub.length > this.MAX_ZOOM) return;
      let k = this.interval / 1000;
      let diff = delta * k * this.sub.length;
      let tl = this.ZOOM_MODE === "tl";
      if (event.originalEvent.ctrlKey || tl) {
        let offset = event.originalEvent.offsetX;
        let diff1 = (offset / (this.canvas.width - 1)) * diff;
        let diff2 = diff - diff1;
        this.range[0] -= diff1;
        this.range[1] += diff2;
      } else {
        this.range[0] -= diff;
      }

      if (tl) {
        let offset = event.originalEvent.offsetY;
        let diff1 = (offset / (this.canvas.height - 1)) * 2;
        let diff2 = 2 - diff1;
        let z = diff / (this.range[1] - this.range[0]);
        //rezoom_range(z, diff_x, diff_y)
        this.$emit("rezoom-range", {
          grid_id: this.id,
          z,
          diff1,
          diff2,
        });
      }

      this.change_range();
    },

    mousedrag(x, y) {
      let dt = (this.drug.t * (this.drug.x - x)) / this.layout.width;

      let d$ = this.layout.$_hi - this.layout.$_lo;
      d$ *= (this.drug.y - y) / this.layout.height;
      let offset = this.drug.o + d$;

      let ls = this.layout.grid.logScale;

      if (ls && this.drug.y_r) {
        let dy = this.drug.y - y;
        var range = this.drug.y_r.slice();
        range[0] = math.exp((0 - this.drug.B + dy) / this.layout.A);
        range[1] = math.exp(
          (this.layout.height - this.drug.B + dy) / this.layout.A
        );
      }

      if (this.drug.y_r && this.y_transform && !this.y_transform.auto) {
        this.$emit("sidebar-transform", {
          grid_id: this.id,
          range: ls
            ? range || this.drug.y_r
            : [this.drug.y_r[0] - offset, this.drug.y_r[1] - offset],
        });
      }

      this.range[0] = this.drug.r[0] + dt;
      this.range[1] = this.drug.r[1] + dt;

      this.change_range();
    },
    pinchzoom(scale) {
      if (scale > 1 && this.sub.length <= this.MIN_ZOOM) return;
      if (scale < 1 && this.sub.length > this.MAX_ZOOM) return;

      let t = this.pinch.t;
      let nt = (t * 1) / scale;

      this.range[0] = this.pinch.r[0] - (nt - t) * 0.5;
      this.range[1] = this.pinch.r[1] + (nt - t) * 0.5;

      this.change_range();
    },
    trackpad_scroll(event) {
      let dt = this.range[1] - this.range[0];

      this.range[0] += event.deltaX * dt * 0.011;
      this.range[1] += event.deltaX * dt * 0.011;

      this.change_range();
    },
    change_range() {
      // TODO: better way to limit the view. Problem:
      // when you are at the dead end of the data,
      // and keep scrolling,
      // the chart continues to scale down a little.
      // Solution: I don't know yet

      if (!this.range.length || this.sub.length < 2) return;

      let l = this.sub.length - 1;
      let data = this.sub;
      let range = this.range;

      range[0] = Utils.clamp(
        range[0],
        -Infinity,
        data[l][0] - this.interval * 5.5
      );

      range[1] = Utils.clamp(
        range[1],
        data[0][0] + this.interval * 5.5,
        Infinity
      );

      // TODO: IMPORTANT scrolling is jerky The Problem caused
      // by the long round trip of 'range-changed' event.
      // First it propagates up to update layout in Chart.vue,
      // then it moves back as watch() update. It takes 1-5 ms.
      // And because the delay is different each time we see
      // the lag. No smooth movement and it's annoying.
      // Solution: we could try to calc the layout immediatly
      // somewhere here. Still will hurt the sidebar & bottombar
      this.$emit("range-changed", range);
    },
    // Propagate mouse event to overlays
    propagate(name, event) {
      for (var layer of this.overlays) {
        if (layer.renderer[name]) {
          layer.renderer[name](event);
        }
        const mouse = layer.renderer.mouse;
        const keys = layer.renderer.keys;
        if (mouse.listeners) {
          mouse.emit(name, event);
        }
        if (keys && keys.listeners) {
          keys.emit(name, event);
        }
      }
    },
  },
};
</script>
