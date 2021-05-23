<script>
export default {
  name: "Overlays",
  props: [
    "overlaysData",
    "layoutGrids",
    "grid_id",
    "overlaysList",
    "registry",
    "meta",

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
    // TODO: see how to use these two functions
    get_overlays(h) {
      // Distributes overlay data & settings according
      // to this._registry; returns compo list
      let comp_list = [],
        count = {};

      for (var d of this.overlaysData) {
        let comp = this.overlaysList[this.registry[d.type]];
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
            grid_id: this.grid_id,
            meta: this.meta,
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
      let f = this.overlaysList.find((x) => x.name === src.conf.renderer);
      if (!f) return comp;

      comp.mixins.push(f);
      comp.__renderer__ = src.conf.renderer;

      return comp;
    },
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
      this.get_overlays(h)
    );
  },
};
</script>
