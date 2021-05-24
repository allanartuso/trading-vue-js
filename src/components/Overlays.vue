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
  data() {
    return {
      layer_events: {},
    };
  },
  watch: {},
  mounted() {
    this.layer_events = {
      "new-grid-layer": (d) => this.$emit("new-grid-layer", d),
      "delete-grid-layer": (d) => this.$emit("delete-grid-layer", d),
      "show-grid-layer": (d) => this.$emit("show-grid-layer", d),
      "redraw-grid": (d) => this.$emit("redraw-grid", d),
      "layer-meta-props": (d) => this.$emit("layer-meta-props", d),
      "custom-event": (d) => this.$emit("custom-event", d),
    };
    this.count = this.overlaysCount;
  },
  methods: {
    common_props() {
      return {
        cursor: this.$props.cursor,
        colors: this.$props.colors,
        layout: this.$props.layoutGrids[this.$props.grid_id],
        interval: this.$props.interval,
        sub: this.$props.sub,
        font: this.$props.font,
        config: this.$props.config,
      };
    },

    createOverlays(h) {
      return (this.overlaysCompList || []).map((x, i) =>
        h(x.cls, {
          on: this.layer_events,
          attrs: Object.assign(this.common_props(), {
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
          }),
        })
      );
    },
    // Replace the current comp with 'renderer'
  },
  render(h) {
    const layout = this.layoutGrids[this.grid_id];
    const props = {
      position: {
        x: 0,
        y: layout.offset || 0,
      },
      attrs: {
        width: layout.width,
        height: layout.height,
        overflow: "hidden",
      },
    };

    const createOverlays = this.createOverlays(h);

    return h(
      "div",
      {
        class: `trading-vue-overlays-${this.grid_id}`,
        style: {
          left: props.position.x + "px",
          top: props.position.y + "px",
          position: "absolute",
        },
      },
      createOverlays
    );
  },
};
</script>
