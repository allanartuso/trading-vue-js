<template> </template>

<script>
export default {
  name: "Crosshair",
  props: ["cursor", "colors", "layout", "sub"],
  data() {
    return {
      x: undefined,
      y: undefined,
      visible: undefined,
      locked: undefined,
    };
  },
  watch: {
    cursor: {
      handler: function() {
        // Explore = default mode on mobile
        const explore = this.cursor.mode === "explore";

        if (!this.cursor.x || !this.cursor.y) {
          this.hide();
          this.$emit("redraw-grid");
          return;
        }
        this.visible = !explore;
      },
      deep: true,
    },
  },
  methods: {
    create() {
      // New grid overlay-renderer descriptor.
      // Should implement draw() (see Spline.vue)
      this.$emit("new-grid-layer", {
        name: "crosshair",
        renderer: this,
      });
    },
    draw(ctx) {
      // Update reference to the grid

      if (!this.visible && this.cursor.mode === "explore") return;

      this.x = this.cursor.x;
      this.y = this.cursor.y;

      ctx.save();
      ctx.strokeStyle = this.colors.cross;
      ctx.beginPath();
      ctx.setLineDash([5]);

      // H
      if (this.cursor.grid_id === this.layout.id) {
        ctx.moveTo(0, this.y);
        ctx.lineTo(this.layout.width - 0.5, this.y);
      }

      // V
      ctx.moveTo(this.x, 0);
      ctx.lineTo(this.x, this.layout.height);
      ctx.stroke();
      ctx.restore();
    },

    hide() {
      this.visible = false;
      this.x = undefined;
      this.y = undefined;
    },
  },
};
</script>
