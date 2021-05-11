<template>
  <!-- Chart components combined together -->
  <grid-base
    ref="renderer"
    :gridClass="`trading-vue-grid-${grid_id}`"
    :cursor="cursor"
    :layoutGrids="layout.grids"
    :grid_id="grid_id"
    :propRange="range"
    :sub="sub"
    :interval="interval"
    :shaders="shaders"
    :colors="colors"
    :meta="meta"
    :y_transform="y_transform"
    :config="config"
    :tv_id="tv_id"
    :font="font"
    :attrs="{ width, height }"
    :position="{
      x: 0,
      y: layout.offset || 0,
    }"
    :SCROLL_WHEEL="config.SCROLL_WHEEL"
    :MIN_ZOOM="config.MIN_ZOOM"
    :MAX_ZOOM="config.MAX_ZOOM"
    :ZOOM_MODE="config.ZOOM_MODE"
    @range-changed="range_changed"
    @cursor-changed="cursor_changed"
    @cursor-locked="cursor_locked"
    @custom-event="emit_custom_event"
    @sidebar-transform="sidebar_transform"
    @rezoom-range="rezoom_range"
  />
</template>

<script>
// Sets up all layers/overlays for the grid with 'grid_id'

import UxList from "../mixins/uxlist.js";
import GridBase from "./GridBase.vue";

import Spline from "./overlays/Spline.vue";
import Splines from "./overlays/Splines.vue";
import Range from "./overlays/Range.vue";
import Trades from "./overlays/Trades.vue";
import Channel from "./overlays/Channel.vue";
import Segment from "./overlays/Segment.vue";
import Candles from "./overlays/Candles.vue";
import Volume from "./overlays/Volume.vue";
import Splitters from "./overlays/Splitters.vue";
import LineTool from "./overlays/LineTool.vue";
import RangeTool from "./overlays/RangeTool.vue";

export default {
  name: "Grid",
  components: { GridBase },
  mixins: [UxList],
  props: [
    "sub",
    "layout",
    "range",
    "interval",
    "cursor",
    "colors",
    "overlays",
    "width",
    "height",
    "data",
    "grid_id",
    "y_transform",
    "font",
    "tv_id",
    "config",
    "meta",
    "shaders",
  ],

  data() {
    return {
      layer_events: {},
      keyboard_events: {
        "register-kb-listener": (event) => {
          this.$emit("register-kb-listener", event);
        },
        "remove-kb-listener": (event) => {
          this.$emit("remove-kb-listener", event);
        },
        keyup: (event) => {
          if (!this.is_active) return;
          if (this.$refs.renderer)
            this.$refs.renderer.propagate("keyup", event);
        },
        keydown: (event) => {
          if (!this.is_active) return; // TODO: is this neeeded?
          if (this.$refs.renderer)
            this.$refs.renderer.propagate("keydown", event);
        },
        keypress: (event) => {
          if (!this.is_active) return;
          if (this.$refs.renderer)
            this.$refs.renderer.propagate("keypress", event);
        },
      },
    };
  },
  computed: {
    is_active() {
      return (
        this.$props.cursor.t !== undefined &&
        this.$props.cursor.grid_id === this.$props.grid_id
      );
    },
  },
  watch: {
    range: {
      handler: function() {
        // TODO: Left-side render lag fix:
        // Overlay data is updated one tick later than
        // the main sub. Fast fix is to delay redraw()
        // call. It will be a solution until a better
        // one comes by.
        this.$nextTick(() => this.redraw());
      },
      deep: true,
    },
    cursor: {
      handler: function() {
        if (!this.$props.cursor.locked) this.redraw();
      },
      deep: true,
    },
    overlays: {
      // Track changes in calc() functions
      handler: function(ovs) {
        for (var ov of ovs) {
          for (var comp of this.$children) {
            if (typeof comp.id !== "string") continue;
            let tuple = comp.id.split("_");
            tuple.pop();
            if (tuple.join("_") === ov.name) {
              comp.calc = ov.methods.calc;
              if (!comp.calc) continue;
              let calc = comp.calc.toString();
              if (calc !== ov.__prevscript__) {
                comp.exec_script();
              }
              ov.__prevscript__ = calc;
            }
          }
        }
      },
      deep: true,
    },
    // Redraw on the shader list change
    shaders(n, p) {
      this.redraw();
    },
    width(val) {
      if (this.$refs.renderer) this.$refs.renderer.setup();
    },
    height(val) {
      if (this.$refs.renderer) this.$refs.renderer.setup();
    },
  },

  created() {
    // List of all possible overlays (builtin + custom)
    this._list = [
      Spline,
      Splines,
      Range,
      Trades,
      Channel,
      Segment,
      Candles,
      Volume,
      Splitters,
      LineTool,
      RangeTool,
    ].concat(this.$props.overlays);
    this._registry = {};

    // We need to know which components we will use.
    // Custom overlay components overwrite built-ins:
    var tools = [];
    this._list.forEach((x, i) => {
      let use_for = x.methods.use_for();
      if (x.methods.tool)
        tools.push({
          use_for,
          info: x.methods.tool(),
        });
      use_for.forEach((indicator) => {
        this._registry[indicator] = i;
      });
    });
    this.$emit("custom-event", {
      event: "register-tools",
      args: tools,
    });
    this.$on("custom-event", (e) => this.on_ux_event(e, "grid"));
  },

  mounted() {
    this.layer_events = {
      "new-grid-layer": this.new_layer,
      "delete-grid-layer": this.del_layer,
      "show-grid-layer": (d) => {
        if (this.$refs.renderer) this.$refs.renderer.show_hide_layer(d);
        this.redraw();
      },
      "redraw-grid": this.$refs.renderer.update,
      "layer-meta-props": (d) => this.$emit("layer-meta-props", d),
      "custom-event": (d) => this.$emit("custom-event", d),
    };
  },

  methods: {
    redraw() {
      if (this.$refs.renderer) this.$refs.renderer.update();
    },
    range_changed(r) {
      this.$emit("range-changed", r);
    },
    cursor_changed(c) {
      this.$emit("cursor-changed", c);
    },
    cursor_locked(state) {
      this.$emit("cursor-locked", state);
    },
    sidebar_transform(s) {
      this.$emit("sidebar-transform", s);
    },
    rezoom_range(s) {
      this.$emit("rezoom-range", s);
    },
    emit_custom_event(s) {
      this.$emit("custom-event", s);
    },
    new_layer(layer) {
      this.$nextTick(() => this.$refs.renderer.new_layer(layer));
    },
    del_layer(layer) {
      this.$nextTick(() => this.$refs.renderer.del_layer(layer));
      const grid_id = this.$props.grid_id;
      this.$emit("custom-event", {
        event: "remove-shaders",
        args: [grid_id, layer],
      });
      // TODO: close all interfaces
      this.$emit("custom-event", {
        event: "remove-layer-meta",
        args: [grid_id, layer],
      });
      this.remove_all_ux(layer);
    },
    common_props() {
      return {
        cursor: this.$props.cursor,
        colors: this.$props.colors,
        layout: this.$props.layout.grids[this.$props.grid_id],
        interval: this.$props.interval,
        sub: this.$props.sub,
        font: this.$props.font,
        config: this.$props.config,
      };
    },
    emit_ux_event(e) {
      let e_pass = this.on_ux_event(e, "grid");
      if (e_pass) this.$emit("custom-event", e);
    },

    // TODO: see how to use these two functions
    get_overlays(h) {
      // Distributes overlay data & settings according
      // to this._registry; returns compo list
      let comp_list = [],
        count = {};

      for (var d of this.$props.data) {
        let comp = this._list[this._registry[d.type]];
        if (comp) {
          if (comp.methods.calc) {
            comp = this.inject_renderer(comp);
          }
          comp_list.push({
            cls: comp,
            type: d.type,
            data: d.data,
            settings: d.settings,
            i0: d.i0,
            tf: d.tf,
            last: d.last,
          });
          count[d.type] = 0;
        }
      }
      return comp_list.map((x, i) =>
        h(x.cls, {
          on: this.layer_events,
          attrs: Object.assign(this.common_props(), {
            id: `${x.type}_${count[x.type]++}`,
            type: x.type,
            data: x.data,
            settings: x.settings,
            i0: x.i0,
            tf: x.tf,
            num: i,
            grid_id: this.$props.grid_id,
            meta: this.$props.meta,
            last: x.last,
          }),
        })
      );
    },
    // Replace the current comp with 'renderer'
    inject_renderer(comp) {
      let src = comp.methods.calc();
      if (!src.conf || !src.conf.renderer || comp.__renderer__) {
        return comp;
      }

      // Search for an overlay with the target 'name'
      let f = this._list.find((x) => x.name === src.conf.renderer);
      if (!f) return comp;

      comp.mixins.push(f);
      comp.__renderer__ = src.conf.renderer;

      return comp;
    },
  },
};
</script>
