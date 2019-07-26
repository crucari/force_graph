<template>
  <div
    v-once
    class="fill"
    @contextmenu.prevent
    @mousewheel="test"
    @mousedown="dragged"
    @touchstart="dragged"
    @touchend="dragended"
    @mouseup="dragended"
  >
    <canvas ref="canvas"/>
    <div class="animated pulsate">
      <h1>{{particleAmount}}</h1>
      <h1>Hierarchical Nodes</h1>

      <small>Move and drag your mouse, use mousewheel, weee!</small>
    </div>
  </div>
</template>

<script>
/* eslint-disable no-unused-vars */
import * as pixi from "pixi.js";
import { range } from "d3-array";
import {
  forceSimulation,
  forceManyBody,
  forceLink,
  forceX,
  forceY,
  forceCenter,
  forceRadial
} from "d3-force";
import { event } from "d3-selection";
import {
  scaleSequential,
  interpolateSinebow,
  rgb,
  interpolateSpectral,
  interpolateCubehelix,
  interpolateViridis,
  interpolateRainbow
} from "d3";
export default {
  data() {
    return {
      dragging: false,
      width: null,
      height: null,
      zoom: 10,
      colorScale: null,
      pr: devicePixelRatio,
      particleAmount: pixi.utils.isMobile.any ? 500 : 3500
    };
  },
  computed: {
    /**
     * @type {Vue.ComputedOptions<d3.Simulation<ForceTestNode,ForceTestLink>}
     */
    simulation: {
      get() {
        return this.$options.simulation;
      },
      set(v) {
        this.$options.simulation = v;
      }
    },

    /**
     * @type {Vue.ComputedOptions<PIXI.Application>}
     */
    pixiRender: {
      get() {
        return this.$options.pixi;
      },
      set(v) {
        this.$options.pixi = v;
      }
    }
  },
  beforeDestroy() {
    this.simulation.stop();
    this.simulation = null;

    this.pixiRender.destroy(true);

    this.$delete(window, "app");

    window.removeEventListener("resize", this.onResize);
    window.removeEventListener("mousemove", this.updateMouse);
  },
  mounted() {
    const { width, height } = this.$el.getBoundingClientRect();

    /** @type {PIXI.RendererOptions} */
    const options = {
      antialias: false,
      // preserveDrawingBuffer: true,
      roundPixels: true,
      powerPreference: "high-performance",
      resolution: devicePixelRatio,
      transparent: true,
      forceFXAA: true
    };

    const r = devicePixelRatio;

    pixi.utils.skipHello();

    this.pixiRender = new pixi.Application({
      view: this.$refs.canvas,
      // powerPreference: 'high-performance',
      // forceFXAA: true,
      width,
      height,
      ...options
    });

    // Object.assign(this.pixiRender.renderer.options, options)

    /** @type {PIXI.Sprite} */
    // const bg = new pixi.Sprite.fromImage('/bg.png')

    // bg.width = width
    // bg.height = height
    // this.pixiRender.stage.addChild(bg)

    window.app = this.pixiRender;

    Object.assign(this.$data, {
      width,
      height
    });

    const container = new pixi.particles.ParticleContainer(
      10000,
      {
        vertices: false,
        rotation: false,
        scale: false,
        uvs: false,
        alpha: false,
        tint: false,
        position: true
      },
      200
    );

    // container.position.set(width / 2, height / 2)

    container.interactive = false;
    container.interactiveChildren = false;
    // container.blendMode = pixi.BLEND_MODES.OVERLAY

    this.particleAmount = pixi.utils.isMobile.any
      ? 500
      : pixi.utils.isMobile.any
      ? 1000
      : this.particleAmount;

    const roots = range(~~Math.sqrt(this.particleAmount));

    this.pixiRender.stage.addChild(container);

    this.colorScale = scaleSequential(interpolateRainbow).domain([
      0,
      this.particleAmount
    ]);

    this.nodes = range(this.particleAmount).map((k, i) => {
      /** @type {PIXI.Sprite} */
      const spr = new pixi.Sprite.fromImage("/test.png");
      container.addChild(spr);

      // spr.visible = roots.includes(i+1)

      if (roots.includes(i)) {
        // console.log('wtf')
        spr.alpha = 0;
      }

      const [r, g, b] = this.colorScale(i)
        .replace(/[^\d,]/g, "")
        .split(",")
        .map(v => +v);

      spr.tint = parseInt(
        rgb(r, g, b)
          .hex()
          .substr(1),
        16
      );

      return {
        index: i,
        spr,
        ...spr.position,
        color: this.colorScale(Math.sqrt(i))
      };
    });

    this.links = range(this.nodes.length - 1).map(i => ({
      source: Math.floor(Math.sqrt(i)),
      target: i + 1
    }));

    this.simulation = forceSimulation(this.nodes)
      .force("charge", forceManyBody().strength(-60))
      .force("link", forceLink(this.links).distance(this.zoom))
      .force(
        "radial",
        forceRadial(height / 1.4, width / 2, height / 2).strength(0.51)
      )
      .force("mouse", forceRadial(100, width / 2, height / 2).strength(1))
      .force("center", forceCenter(width / 2, height / 2))

      // .force('x', forceX())
      // .force('y', forceY())
      .on("tick", this.ticked)
      .alphaDecay(0.16)
      .alpha(0.5)
      .alphaTarget(0.1)
      .tick(100);

    window.addEventListener("resize", this.onResize);
    window.addEventListener("mousemove", this.updateMouse);
    window.addEventListener("touchmove", this.updateTouch);
  },

  methods: {
    /**
     * @type {(e: MouseWheelEvent) => void}
     */
    test(e) {
      this.zoom += e.deltaY * 0.5;

      this.simulation
        .restart()
        .alphaDecay(0.0004)
        .alpha();
    },
    ticked() {
      this.simulation.force("link").distance(this.zoom);

      // return
      this.nodes.forEach((n, i) => {
        const { x, y, spr } = n;

        spr.position.set(x, y);
      });
    },

    dragsubject() {
      const { width, height } = this;
      return this.simulation.find(event.x - width / 2, event.y - height / 2);
    },
    dragstarted() {
      this.simulation.alphaTarget(0.5).restart();
      this.dragging = true;
    },

    /**
     * @param {MouseEvent} e
     */
    dragged() {
      this.simulation.alphaTarget(0.5).restart();
      this.dragging = true;
      if (!this.dragging) return;
      /** @type {d3.ForceManyBody} */
      const { strength } = this.simulation.force("charge");

      strength(-25);

      /** @type {d3.ForceLink} */
      const f = this.simulation.force("link");
      f.strength(0.01);
    },
    dragended() {
      this.dragging = false;
      this.simulation.alphaTarget(0.1).alpha(0.5);
      /** @type {d3.ForceManyBody} */
      const { strength } = this.simulation.force("charge");

      strength(-60);

      /** @type {d3.ForceLink} */
      const f = this.simulation.force("link");
      f.strength(0.7);
    },

    /**
     * @param {d3.SimulationLinkDatum<ForceTestNode>} d
     */
    drawLink(d) {
      /** @type {pixi.Graphics} */
      const g = this.pixiRender.stage.children[0];
      g.moveTo(d.source.x, d.source.y).lineTo(d.target.x, d.target.y);
    },

    onResize() {
      this.width = innerWidth;
      this.height = innerHeight;
      this.pixiRender.renderer.resize(innerWidth, innerHeight);

      /** @type {PIXI.particles.ParticleContainer} */
      const container = this.pixiRender.stage.children[0];
      // container.position.set(this.width / 2, this.height / 2);
      container.position.set(0, 0);
    },

    /** @param {MouseEvent} e */
    updateMouse(e) {
      const { x, y } = e;

      /** @type {d3.ForceRadial} */
      const force = this.simulation.force("mouse");

      force.x(x).y(y);
    },

    /** @param {TouchEvent} e */
    updateTouch(e) {
      const { pageX: x, pageY: y } = e.touches[0];

      /** @type {d3.ForceRadial} */
      const force = this.simulation.force("mouse");

      force.x(x).y(y);
    }
  }
};
</script>

<style scoped lang="scss">
small {
  margin-top: 20px;
  font-style: italic;
  font-weight: 200;
  font-size: 12px;
  color: rgba(#fff, 0.5);
  text-transform: lowercase;
}
canvas {
  width: 100%;
  height: 100%;
  background-image: radial-gradient(#2b414f, #20313a);

  + div {
    top: 0;
    color: #fff;
    left: 0;
    user-select: none;
    width: 100vw;
    height: 100vh;
    position: absolute;
    display: flex;
    flex-flow: column nowrap;
    align-items: center;
    justify-content: center;
    transform: translateZ(0);

    > * {
      display: block;
      /* margin: 0; */
      text-transform: uppercase;

      max-width: 300px;

      &:first-child {
        font-size: 60px;
        font-weight: bold;
        margin-bottom: 10px;
        color: #ffccff;
      }
    }
  }
}
</style>
