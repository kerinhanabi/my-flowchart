<template>
    <div class="flowchart-container" 
        @mousemove="handleMove" 
        @mouseup="handleUp"
        @mousedown="handleDown">
        <svg width="100%" :height="`${height}px`">
        <flowchart-link :link="link" 
            v-for="(link, index) in lines" 
            :key="`link${index}`"
            @deleteLink="linkDelete(link.id)">
        </flowchart-link>
        </svg>

        <flowchart-node
        v-for="(node, index) in scene.nodes" 
        :key="`node${index}`" 
        :node="node" 
        :options="nodeOptions"
        :x="node.x" :y="node.y"
        @linkingStart="linkingStart(node.id)" 
        @linkingStop="linkingStop(node.id)" 
        @nodeSelected="nodeSelected(node.id, $event)" 
        @nodeDelete="nodeDelete(node.id)">
        </flowchart-node>
    </div>
</template>

<script>
import FlowchartLink from './FlowchartLink.vue';
import FlowchartNode from './FlowchartNode.vue';
import { getMousePosition } from '../assets/utility/position.js';

export default {
    name: 'VueFlowchart',
    props: {
      scene: {
          type: Object,
          default() {
              return {
                  centerX: 1024,
                  scale: 1,
                  centerY: 140,
                  nodes: [],
                  links: [],
                  x: Math.random() * 500,
                  y: Math.random() * 500

              }
          }
      },
      height: {
          type: Number,
          default: 400,
      },
    },
    data() {
      return {
        localScene: JSON.parse(JSON.stringify(this.scene)),
        dragging: false,
        currentNode: null,
        offsetX: 0,
        offsetY: 0,
        action: {
          linking: false,
          dragging: false,
          scrolling: false,
          selected: 0,
        },
        mouse: {
          x: 0,
          y: 0,
          lastX: 0,
          lastY: 0,
        },
        draggingLink: null,
        rootDivOffset: {
          top: 0,
          left: 0
        },
      };
    },
    components: {
      FlowchartLink,
      FlowchartNode,
    },
    computed: {
      nodeOptions() {
        return {
          centerY: this.localScene.centerY,
          centerX: this.localScene.centerX,
          scale: this.localScene.scale,
          offsetTop: this.rootDivOffset.top,
          offsetLeft: this.rootDivOffset.left,
          selected: this.action.selected,
        }
      },
      lines() {
        const lines = [];
        this.localScene.links.forEach(link => {
          const fromNode = this.localScene.nodes.find(node => node.id === link.from);
          const toNode = this.localScene.nodes.find(node => node.id === link.to);
          if (!fromNode || !toNode) {
            console.error(`Nodes not found for link ${link.id}`);
            return;
          }
          let x, y, cy, cx, ex, ey;
          x = this.localScene.centerX + fromNode.x;
          y = this.localScene.centerY + fromNode.y;
          [cx, cy] = this.getPortPosition('bottom', x, y);
          x = this.localScene.centerX + toNode.x;
          y = this.localScene.centerY + toNode.y;
          [ex, ey] = this.getPortPosition('top', x, y);
          lines.push({ 
            start: [cx, cy], 
            end: [ex, ey],
            id: link.id,
          });
        });
        return lines;
      },
    },
    mounted() {
      this.rootDivOffset.top = this.$el ? this.$el.offsetTop : 0;
      this.rootDivOffset.left = this.$el ? this.$el.offsetLeft : 0;
      this.localScene.nodes.forEach(node => {
        node.x = Math.random() * (this.$el.clientWidth - 80);
        node.y = Math.random() * (this.height - 80);
      });
    },
    methods: {
      findNodeWithID(id) {
        return this.localScene.nodes.find((item) => {
            return id === item.id
        })
      },
      getPortPosition(type, x, y) {
        if (type === 'top') {
          return [x + 40, y];
        }
        else if (type === 'bottom') {
          return [x + 40, y + 80];
        }
      },
      linkingStart(index) {
        this.action.linking = true;
        this.draggingLink = {
          from: index,
          mx: 0,
          my: 0,
        };
      },
      linkingStop(index) {
        // add new Link
        if (this.draggingLink && this.draggingLink.from !== index) {
          // check link existence
          const existed = this.localScene.links.find((link) => {
            return link.from === this.draggingLink.from && link.to === index;
          })
          if (!existed) {
            let maxID = Math.max(0, ...this.localScene.links.map((link) => {
              return link.id
            }))
            const newLink = {
              id: maxID + 1,
              from: this.draggingLink.from,
              to: index,
            };
            this.localScene.links.push(newLink)
            this.$emit('linkAdded', newLink)
          }
        }
        this.draggingLink = null
      },
      linkDelete(id) {
        const deletedLink = this.localScene.links.find((item) => {
            return item.id === id;
        });
        if (deletedLink) {
          this.localScene.links = this.localScene.links.filter((item) => {
              return item.id !== id;
          });
          this.$emit('linkBreak', deletedLink);
        }
      },
      nodeSelected(id, e) {
        this.action.dragging = id;
        this.action.selected = id;
        this.$emit('nodeClick', id);
        this.mouse.lastX = e.pageX || e.clientX + document.documentElement.scrollLeft
        this.mouse.lastY = e.pageY || e.clientY + document.documentElement.scrollTop
                
        const nodeIndex = this.localScene.nodes.findIndex(node => node.id === id);
        if (nodeIndex !== -1) {
          this.localScene.nodes[nodeIndex].visible = true;
        }
      },
      handleMove(e) {
        if (this.action.linking) {
          [this.mouse.x, this.mouse.y] = getMousePosition(this.$el, e);
          [this.draggingLink.mx, this.draggingLink.my] = [this.mouse.x, this.mouse.y];
        }
        if (this.action.dragging) {
          this.mouse.x = e.pageX || e.clientX + document.documentElement.scrollLeft
          this.mouse.y = e.pageY || e.clientY + document.documentElement.scrollTop
          let diffX = this.mouse.x - this.mouse.lastX;
          let diffY = this.mouse.y - this.mouse.lastY;
  
          this.mouse.lastX = this.mouse.x;
          this.mouse.lastY = this.mouse.y;
          this.moveSelectedNode(diffX, diffY);
        }
        if (this.action.scrolling) {
          [this.mouse.x, this.mouse.y] = getMousePosition(this.$el, e);
          let diffX = this.mouse.x - this.mouse.lastX;
          let diffY = this.mouse.y - this.mouse.lastY;
  
          this.mouse.lastX = this.mouse.x;
          this.mouse.lastY = this.mouse.y;
  
          this.localScene.centerX += diffX;
          this.localScene.centerY += diffY;
  
          // this.hasDragged = true
        }
      },
      handleUp(e) {
        const target = e.target || e.srcElement;
        if (this.$el.contains(target)) {
          if (typeof target.className !== 'string' || target.className.indexOf('node-input') < 0) {
            this.draggingLink = null;
          }
          if (typeof target.className === 'string' && target.className.indexOf('node-delete') > -1) {
            // console.log('delete2', this.action.dragging);
            this.nodeDelete(this.action.dragging);
          }
        }
        this.action.linking = false;
        this.action.dragging = null;
        this.action.scrolling = false;
      },
      handleDown(e) {
        const target = e.target || e.srcElement;
        // console.log('for scroll', target, e.keyCode, e.which)
        if ((target === this.$el || target.matches('svg, svg *')) && e.which === 1) {
          this.action.scrolling = true;
          [this.mouse.lastX, this.mouse.lastY] = getMousePosition(this.$el, e);
          this.action.selected = null; // deselectAll
        }
        this.$emit('canvasClick', e);
      },

      moveSelectedNode(dx, dy) {
        let index = this.localScene.nodes.findIndex((item) => {
          return item.id === this.action.dragging
        })
        if (index !== -1) {
          let left = this.localScene.nodes[index].x + dx / this.localScene.scale;
          let top = this.localScene.nodes[index].y + dy / this.localScene.scale;
          this.localScene.nodes[index] = Object.assign(this.localScene.nodes[index], {
            x: left,
            y: top,
          });
        }
      },
      nodeDelete(id) {
        const nodeIndex = this.localScene.nodes.findIndex(node => node.id === id);
        if (nodeIndex !== -1) {
          this.localScene.nodes.splice(nodeIndex, 1);
          this.$emit('nodeDelete', id);
        }
        this.localScene.nodes = this.localScene.nodes.filter((node) => {
          return node.id !== id;
        })
        this.localScene.links = this.localScene.links.filter((link) => {
          return link.from !== id && link.to !== id
        })
        this.$emit('nodeDelete', id)
      },

      handleMousedown(event) {
        const id = event.target.dataset.id;
        if (id) {
          this.dragging = true;
          this.currentNode = this.localScene.nodes.find((node) => node.id === parseInt(id));
          this.offsetX = event.clientX - this.currentNode.x;
          this.offsetY = event.clientY - this.currentNode.y;
        }
      },
      handleMouseMove(event) {
        if (this.dragging) {
          this.currentNode.x = event.clientX - this.offsetX;
          this.currentNode.y = event.clientY - this.offsetY;
        }
      },
      handleMouseUp(event) {
        console.log('Mouse up event triggered');
        console.log('Node removed:', this.localScene.nodes);
        this.dragging = false;
        this.currentNode = null;
        if (event.target.classList.contains('node-delete')) {
          const id = event.target.parentNode.dataset.id;
          this.localScene.nodes = this.localScene.nodes.filter((node) => node.id !== parseInt(id));
          this.$emit('nodeDelete', id);
        }
      },

    },
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
.flowchart-container {
margin: 0;
background: #ddd;
position: relative;
overflow: hidden;
    svg {
        cursor: grab;
    }
}
</style>