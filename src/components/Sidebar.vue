<template>
  <div :class="divClass" :style="divStyle">
    <canvas
      :id="canvasId"
      ref="canvas"
      :style="canvasStyle"
      @mousemove="mousemove"
      @mouseout="mouseout"
      @mouseup="mouseup"
      @mousedown="mousedown"
    />
  </div>
</template>

<script>
// The side bar (yep, that thing with a bunch of $$$)

import * as Hammer from "hammerjs";
import Utils from "../stuff/utils.js";
import math from "../stuff/math.js";

export default {
  name: "Sidebar",
  mixins: [],
  props: [
    "sub",
    "layout",
    "range",
    "interval",
    "cursor",
    "colors",
    "font",
    "width",
    "height",
    "grid_id",
    "rerender",
    "y_transform",
    "tv_id",
    "config",
    "shaders",
  ],
  data() {
    return {
      canvas: undefined,
      ctx: undefined,
      PANHEIGHT: undefined,
      layoutGrid: {},
      side: "right",
      divClass: undefined,
      divStyle: {},
      canvasStyle: {},
      sidebarId: undefined,
      canvasId: undefined,
      canvasHeight: undefined,
    };
  },
  watch: {
    range: {
      handler: function() {
        this.update();
      },
      deep: true,
    },
    cursor: {
      handler: function() {
        this.update();
      },
      deep: true,
    },
    rerender() {
      this.$nextTick(() => this.update());
    },
    width() {
      this.setup();
    },
    height(val) {
      this.canvasHeight = val;
      this.setup();
    },
  },
  created() {
    this.PANHEIGHT = this.config.PANHEIGHT;
    this.sidebarId = `sidebar-${this.grid_id}`;
    this.divClass = `trading-vue-${this.sidebarId}`;
    this.layoutGrid = this.$props.layout.grids[this.grid_id];
    this.divStyle = {
      left: this.layoutGrid.width + "px",
      top: (this.layoutGrid.offset || 0) + "px",
      position: "absolute",
    };

    this.canvasStyle = {
      backgroundColor: this.$props.colors.back,
    };
    this.canvasId = `${this.$props.tv_id}-${this.sidebarId}-canvas`;
    this.canvasHeight = this.layoutGrid.height;
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
    if (this.mc) this.mc.destroy();
  },
  methods: {
    setup() {
      let dpr = window.devicePixelRatio || 1;
      this.canvas.style.width = `${this.width}px`;
      this.canvas.style.height = `${this.canvasHeight}px`;
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
        ctx.measureText = (text) =>
          Utils.measureText(ctx, text, this.$props.tv_id);
      });
    },

    listeners() {
      let mc = (this.mc = new Hammer.Manager(this.canvas));
      mc.add(
        new Hammer.Pan({
          direction: Hammer.DIRECTION_VERTICAL,
          threshold: 0,
        })
      );

      mc.add(
        new Hammer.Tap({
          event: "doubletap",
          taps: 2,
          posThreshold: 50,
        })
      );

      mc.on("panstart", (event) => {
        if (this.$props.y_transform) {
          this.zoom = this.$props.y_transform.zoom;
        } else {
          this.zoom = 1.0;
        }
        this.y_range = [this.layoutGrid.$_hi, this.layoutGrid.$_lo];
        this.drug = {
          y: event.center.y,
          z: this.zoom,
          mid: math.log_mid(this.y_range, this.layoutGrid.height),
          A: this.layoutGrid.A,
          B: this.layoutGrid.B,
        };
      });

      mc.on("panmove", (event) => {
        if (this.drug) {
          this.zoom = this.calc_zoom(event);
          this.$emit("sidebar-transform", {
            grid_id: this.grid_id,
            zoom: this.zoom,
            auto: false,
            range: this.calc_range(),
            drugging: true,
          });
          this.update();
        }
      });

      mc.on("panend", () => {
        this.drug = null;
        this.$emit("sidebar-transform", {
          grid_id: this.grid_id,
          drugging: false,
        });
      });

      mc.on("doubletap", () => {
        this.$emit("sidebar-transform", {
          grid_id: this.grid_id,
          zoom: 1.0,
          auto: true,
        });
        this.zoom = 1.0;
        this.update();
      });

      // TODO: Do later for mobile version
    },

    update() {
      // Update reference to the grid

      var points = this.layoutGrid.ys;
      var x,
        y,
        w,
        h,
        side = this.side;
      var sb = this.layoutGrid.sb;

      //this.ctx.fillStyle = this.$props.colors.back
      this.ctx.font = this.$props.font;

      switch (side) {
        case "left":
          x = 0;
          y = 0;
          w = Math.floor(sb);
          h = this.layoutGrid.height;

          //this.ctx.fillRect(x, y, w, h)
          this.ctx.clearRect(x, y, w, h);

          this.ctx.strokeStyle = this.$props.colors.scale;

          this.ctx.beginPath();
          this.ctx.moveTo(x + 0.5, 0);
          this.ctx.lineTo(x + 0.5, h);
          this.ctx.stroke();

          break;
        case "right":
          x = 0;
          y = 0;
          w = Math.floor(sb);
          h = this.layoutGrid.height;
          //this.ctx.fillRect(x, y, w, h)
          this.ctx.clearRect(x, y, w, h);

          this.ctx.strokeStyle = this.$props.colors.scale;

          this.ctx.beginPath();
          this.ctx.moveTo(x + 0.5, 0);
          this.ctx.lineTo(x + 0.5, h);
          this.ctx.stroke();
          break;
      }

      this.ctx.fillStyle = this.$props.colors.text;
      this.ctx.beginPath();

      for (var p of points) {
        if (p[0] > this.layoutGrid.height) continue;

        var x1 = side === "left" ? w - 0.5 : x - 0.5;
        var x2 = side === "left" ? x1 - 4.5 : x1 + 4.5;

        this.ctx.moveTo(x1, p[0] - 0.5);
        this.ctx.lineTo(x2, p[0] - 0.5);

        var offst = side === "left" ? -10 : 10;
        this.ctx.textAlign = side === "left" ? "end" : "start";
        let d = this.layoutGrid.prec;
        this.ctx.fillText(p[1].toFixed(d), x1 + offst, p[0] + 4);
      }

      this.ctx.stroke();

      if (this.$props.grid_id) this.upper_border();

      this.apply_shaders();

      if (this.$props.cursor.y && this.$props.cursor.y$) this.panel();
    },

    apply_shaders() {
      let layout = this.$props.layout.grids[this.grid_id];
      let props = {
        layout: layout,
        cursor: this.$props.cursor,
      };
      for (var s of this.$props.shaders) {
        this.ctx.save();
        s.draw(this.ctx, props);
        this.ctx.restore();
      }
    },

    upper_border() {
      this.ctx.strokeStyle = this.$props.colors.scale;
      this.ctx.beginPath();
      this.ctx.moveTo(0, 0.5);
      this.ctx.lineTo(this.layoutGrid.width, 0.5);
      this.ctx.stroke();
    },

    // A gray bar behind the current price
    panel() {
      if (this.$props.cursor.grid_id !== this.layoutGrid.id) {
        return;
      }

      let lbl = this.$props.cursor.y$.toFixed(this.layoutGrid.prec);
      this.ctx.fillStyle = this.$props.colors.panel;

      let panwidth = this.layoutGrid.sb + 1;

      let x = -0.5;
      let y = this.$props.cursor.y - this.PANHEIGHT * 0.5 - 0.5;
      let a = 7;
      this.ctx.fillRect(x - 0.5, y, panwidth, this.PANHEIGHT);
      this.ctx.fillStyle = this.$props.colors.textHL;
      this.ctx.textAlign = "left";
      this.ctx.fillText(lbl, a, y + 15);
    },

    calc_zoom(event) {
      let d = this.drug.y - event.center.y;
      let speed = d > 0 ? 3 : 1;
      let k = 1 + (speed * d) / this.layoutGrid.height;
      return Utils.clamp(this.drug.z * k, 0.005, 100);
    },

    // Not the best place to calculate y-range but
    // this is the simplest solution I found up to
    // date
    calc_range(diff1 = 1, diff2 = 1) {
      let z = this.zoom / this.drug.z;
      let zk = (1 / z - 1) / 2;

      let range = this.y_range.slice();
      let delta = range[0] - range[1];

      if (!this.layoutGrid.grid.logScale) {
        range[0] = range[0] + delta * zk * diff1;
        range[1] = range[1] - delta * zk * diff2;
      } else {
        let px_mid = this.layoutGrid.height / 2;
        let new_hi = px_mid - px_mid * (1 / z);
        let new_lo = px_mid + px_mid * (1 / z);

        // Use old mapping to get a new range
        let f = (y) => math.exp((y - this.drug.B) / this.drug.A);

        range[0] = f(new_hi);
        range[1] = f(new_lo);
      }

      return range;
    },

    rezoom_range(delta, diff1, diff2) {
      if (!this.$props.y_transform || this.$props.y_transform.auto) return;

      this.zoom = 1.0;
      // TODO: further work (improve scaling ratio)
      if (delta < 0) delta /= 3.75; // Btw, idk why 3.75, but it works
      delta *= 0.25;
      this.y_range = [this.layoutGrid.$_hi, this.layoutGrid.$_lo];
      this.drug = {
        y: 0,
        z: this.zoom,
        mid: math.log_mid(this.y_range, this.layoutGrid.height),
        A: this.layoutGrid.A,
        B: this.layoutGrid.B,
      };
      this.zoom = this.calc_zoom({
        center: {
          y: delta * this.layoutGrid.height,
        },
      });

      this.$emit("sidebar-transform", {
        grid_id: this.grid_id,
        zoom: this.zoom,
        auto: false,
        range: this.calc_range(diff1, diff2),
        drugging: true,
      });
      this.drug = null;
      this.$emit("sidebar-transform", {
        grid_id: this.grid_id,
        drugging: false,
      });
    },

    mousemove() {},
    mouseout() {},
    mouseup() {},
    mousedown() {},
  },
};
</script>
