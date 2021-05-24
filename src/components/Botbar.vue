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
// The bottom bar (yep, that thing with a bunch of dates)

import Utils from "../stuff/utils.js";
import Const from "../stuff/constants.js";

const { MINUTE15, MINUTE, HOUR, DAY, WEEK, MONTH, YEAR, MONTHMAP } = Const;

export default {
  name: "Botbar",
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
    "rerender",
    "tv_id",
    "config",
    "shaders",
    "timezone",
  ],
  computed: {
    bot_shaders() {
      return this.$props.shaders.filter((x) => x.target === "botbar");
    },
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
    width(val) {
      this._attrs.width = val;
      this.setup();
    },
    height(val) {
      this._attrs.height = val;
      this.setup();
    },
  },
  created() {
    this.sidebarId = `botbar`;
    this.divClass = `trading-vue-${this.sidebarId}`;
    this.divStyle = {
      left: 0 + "px",
      top: (this.layout.botbar.offset || 0) + "px",
      position: "absolute",
    };

    this.canvasStyle = {
      backgroundColor: this.$props.colors.back,
    };
    this.canvasId = `${this.$props.tv_id}-${this.sidebarId}-canvas`;
    this.canvasHeight = this.layout.botbar.height;
  },
  mounted() {
    this.canvas = this.$refs.canvas;
    this.ctx = this.canvas.getContext("2d");

    this.$nextTick(() => {
      this.setup();
    });
  },
  methods: {
    setup() {
      let dpr = window.devicePixelRatio || 1;
      this.canvas.style.width = `${this.layout.botbar.width}px`;
      this.canvas.style.height = `${this.layout.botbar.height}px`;
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

    mousemove() {},
    mouseout() {},
    mouseup() {},
    mousedown() {},

    update() {
      this.grid_0 = this.layout.grids[0];

      const width = this.layout.botbar.width;
      const height = this.layout.botbar.height;

      const sb = this.layout.grids[0].sb;

      //this.ctx.fillStyle = this.$props.colors.back
      this.ctx.font = this.$props.font;
      //this.ctx.fillRect(0, 0, width, height)
      this.ctx.clearRect(0, 0, width, height);

      this.ctx.strokeStyle = this.$props.colors.scale;

      this.ctx.beginPath();
      this.ctx.moveTo(0, 0.5);
      this.ctx.lineTo(Math.floor(width + 1), 0.5);
      this.ctx.stroke();

      this.ctx.fillStyle = this.$props.colors.text;
      this.ctx.beginPath();

      for (var p of this.layout.botbar.xs) {
        let lbl = this.format_date(p);

        if (p[0] > width - sb) continue;

        this.ctx.moveTo(p[0] - 0.5, 0);
        this.ctx.lineTo(p[0] - 0.5, 4.5);

        if (!this.lbl_highlight(p[1][0])) {
          this.ctx.globalAlpha = 0.85;
        }
        this.ctx.textAlign = "center";
        this.ctx.fillText(lbl, p[0], 18);
        this.ctx.globalAlpha = 1;
      }

      this.ctx.stroke();
      this.apply_shaders();
      if (this.$props.cursor.x && this.$props.cursor.t !== undefined)
        this.panel();
    },

    apply_shaders() {
      let layout = this.layout.grids[0];
      let props = {
        layout: layout,
        cursor: this.$props.cursor,
      };
      for (var s of this.bot_shaders) {
        this.ctx.save();
        s.draw(this.ctx, props);
        this.ctx.restore();
      }
    },

    panel() {
      let lbl = this.format_cursor_x();
      this.ctx.fillStyle = this.$props.colors.panel;

      let measure = this.ctx.measureText(lbl + "    ");
      let panwidth = Math.floor(measure.width);
      let cursor = this.$props.cursor.x;
      let x = Math.floor(cursor - panwidth * 0.5);
      let y = -0.5;
      let panheight = this.config.PANHEIGHT;
      this.ctx.fillRect(x, y, panwidth, panheight + 0.5);

      this.ctx.fillStyle = this.$props.colors.textHL;
      this.ctx.textAlign = "center";
      this.ctx.fillText(lbl, cursor, y + 16);
    },

    format_date(p) {
      let t = p[1][0];
      t = this.grid_0.ti_map.i2t(t);
      let ti = this.$props.layout.grids[0].ti_map.tf;
      // Enable timezones only for tf < 1D
      let k = ti < DAY ? 1 : 0;
      let tZ = t + k * this.$props.timezone * HOUR;

      //t += new Date(t).getTimezoneOffset() * MINUTE
      let d = new Date(tZ);

      if (p[2] === YEAR || Utils.year_start(t) === t) {
        return d.getUTCFullYear();
      }
      if (p[2] === MONTH || Utils.month_start(t) === t) {
        return MONTHMAP[d.getUTCMonth()];
      }
      // TODO(*) see grid_maker.js
      if (Utils.day_start(tZ) === tZ) return d.getUTCDate();

      let h = Utils.add_zero(d.getUTCHours());
      let m = Utils.add_zero(d.getUTCMinutes());
      return h + ":" + m;
    },

    format_cursor_x() {
      let t = this.$props.cursor.t;
      t = this.grid_0.ti_map.i2t(t);
      //let ti = this.$props.interval
      let ti = this.$props.layout.grids[0].ti_map.tf;
      // Enable timezones only for tf < 1D
      let k = ti < DAY ? 1 : 0;

      //t += new Date(t).getTimezoneOffset() * MINUTE
      let d = new Date(t + k * this.$props.timezone * HOUR);

      if (ti === YEAR) {
        return d.getUTCFullYear();
      }

      if (ti < YEAR) {
        var yr = "`" + `${d.getUTCFullYear()}`.slice(-2);
        var mo = MONTHMAP[d.getUTCMonth()];
        var dd = "01";
      }
      if (ti <= WEEK) dd = d.getUTCDate();
      let date = `${dd} ${mo} ${yr}`;
      let time = "";

      if (ti < DAY) {
        let h = Utils.add_zero(d.getUTCHours());
        let m = Utils.add_zero(d.getUTCMinutes());
        time = h + ":" + m;
      }

      return `${date}  ${time}`;
    },

    // Highlights the begining of a time interval
    // TODO: improve. Problem: let's say we have a new month,
    // but if there is no grid line in place, there
    // will be no month name on t-axis. Sad.
    // Solution: manipulate the grid, skew it, you know
    lbl_highlight(t) {
      let ti = this.$props.interval;

      if (t === 0) return true;
      if (Utils.month_start(t) === t) return true;
      if (Utils.day_start(t) === t) return true;
      if (ti <= MINUTE15 && t % HOUR === 0) return true;

      return false;
    },
  },
};
</script>
<style>
.trading-vue-botbar {
  position: relative !important;
}
</style>
