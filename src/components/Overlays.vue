<script>
export default {
  name: "Overlays",
  props: [
    "layoutGrids",
    "grid_id",
    "overlaysCompList",
    "meta",
    "overlaysCount",
    "cursor",
    "colors",
    "interval",
    "sub",
    "font",
    "config",
  ],
  created() {
    this.count = this.overlaysCount;
  },
  render(h) {
    return h(
      "div",
      {},
      (this.overlaysCompList || []).map((x, i) => {
        return h(x.cls, {
          on: {
            "new-grid-layer": (d) => this.$emit("new-grid-layer", d),
            "delete-grid-layer": (d) => this.$emit("delete-grid-layer", d),
            "show-grid-layer": (d) => this.$emit("show-grid-layer", d),
            "redraw-grid": (d) => this.$emit("redraw-grid", d),
            "layer-meta-props": (d) => this.$emit("layer-meta-props", d),
            "custom-event": (d) => this.$emit("custom-event", d),
          },
          attrs: {
            cursor: this.cursor,
            colors: this.colors,
            layout: this.layoutGrids[this.grid_id],
            interval: this.interval,
            sub: this.sub,
            font: this.font,
            config: this.config,
            id: `${x.type}_${this.count[x.type]++}`,
            type: x.type,
            data: x.data,
            settings: x.settings,
            i0: x.i0,
            tf: x.tf,
            num: i,
            grid_id: this.grid_id,
            meta: this.meta,
            last: x.last,
          },
        });
      })
    );
  },
};
</script>
