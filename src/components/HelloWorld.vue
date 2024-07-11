<template>
  <div ref="goJsContainer" style="width: 600px; height: 400px; border: 1px solid black;"></div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted, toRefs, defineProps } from 'vue';
import * as go from 'gojs';
import {NodeData,LinkData,Rect} from "./type"
const props = defineProps<{
  nodes: Array<NodeData>,
  links: Array<LinkData>,
  restrictedArea: Rect
}>();

const { nodes, links, restrictedArea } = toRefs(props);

const goJsContainer = ref<HTMLDivElement | null>(null);
let diagram: go.Diagram;

const initDiagram = () => {
  const $ = go.GraphObject.make;
  diagram = $(go.Diagram, goJsContainer.value!, {
    
    "undoManager.isEnabled": true
  });

  diagram.nodeTemplate = $(go.Node, "Auto",
    $(go.Shape, "RoundedRectangle",
      { strokeWidth: 0 },
      new go.Binding("fill", "color")),
    $(go.TextBlock,
      { margin: 8 },
      new go.Binding("text", "key"))
  );
  // 绘制红色可移动区域
  const redArea = new go.Part();
  redArea.layerName = "Background";
  redArea.location = new go.Point(0, 0);
  redArea.selectable = false;
  redArea.pickable = false;
  redArea.add(
    $(go.Shape, "RoundedRectangle", { fill: "red", stroke: null, width: 300, height: 200 })
  );
  diagram.add(redArea);
  const gradScaleHoriz =
        new go.Part("Graduated",
            {graduatedMin: 0, graduatedTickUnit: 10, layerName: "ViewportBackground", alignment: go.Spot.TopLeft, isAnimated: false })
          .add(
            new go.Shape({ geometryString: "M0 0 H400" }),
            new go.Shape({ geometryString: "M0 0 V4", interval: 1 }),
            new go.Shape({ geometryString: "M0 0 V7", interval: 5 }),
            new go.Shape({ geometryString: "M0 0 V15", interval: 10 }),
            new go.TextBlock(
              {
                font: "10px sans-serif",
                interval: 10,
                alignmentFocus: go.Spot.TopLeft,
                segmentOffset: new go.Point(0, 12)
              })
          );

      const gradScaleVert =
        new go.Part("Graduated",
            { graduatedTickUnit: 10, layerName: "ViewportBackground", alignment: go.Spot.TopLeft, isAnimated: false })
          .add(
            new go.Shape({ geometryString: "M0 0 V400" }),
            new go.Shape({ geometryString: "M0 0 V4", interval: 1, alignmentFocus: go.Spot.Bottom }),
            new go.Shape({ geometryString: "M0 0 V7", interval: 5, alignmentFocus: go.Spot.Bottom }),
            new go.Shape({ geometryString: "M0 0 V15", interval: 10, alignmentFocus: go.Spot.Bottom }),
            new go.TextBlock(
              {
                font: "10px sans-serif",
                segmentOrientation: go.Orientation.Opposite,
                interval: 10,
                alignmentFocus: go.Spot.BottomLeft,
                segmentOffset: new go.Point(0, -7)
              })
          );
  
      function setupScalesAndIndicators() {
        diagram.commit(d => {
          // Add each node to the diagram
          d.add(gradScaleHoriz);
          d.add(gradScaleVert);
          updateScales();
        }, null);  // null says to skip UndoManager recording of changes
      }

      function updateScales(vb) {
        console.log(vb,'vb')
        if (!vb) vb = diagram.viewportBounds;
        if (!vb.isReal()) return;
        diagram.commit(diag => {
          // Update properties of horizontal scale to reflect viewport
          gradScaleHoriz.elt(0).width = vb.width * diag.scale;
          gradScaleHoriz.graduatedMin = 0;
          gradScaleHoriz.graduatedMax = vb.right;
          // Update properties of vertical scale to reflect viewport
          gradScaleVert.elt(0).height = vb.height * diag.scale;
          gradScaleVert.graduatedMin = 0;
          gradScaleVert.graduatedMax = vb.bottom;
        }, null);
      }
  diagram.addDiagramListener("SelectionMoved", function (e) {
    console.log("log")
    const node = e.subject.first(); // assuming single selection
    if (node) {
      const nodeBounds = node.actualBounds;
      const newPosition = node.position.copy();

      const area = restrictedArea.value;

      const restrictedRect = new go.Rect(area.x, area.y, area.width, area.height);

      // Constrain node within the restricted area
      if (nodeBounds.left < restrictedRect.left) newPosition.x = restrictedRect.left;
      if (nodeBounds.top < restrictedRect.top) newPosition.y = restrictedRect.top;
      if (nodeBounds.right > restrictedRect.right) newPosition.x = restrictedRect.right - nodeBounds.width;
      if (nodeBounds.bottom > restrictedRect.bottom) newPosition.y = restrictedRect.bottom - nodeBounds.height;
      node.position = newPosition;
    }
  });
  diagram.addDiagramListener("InitialLayoutCompleted", setupScalesAndIndicators);
  updateDiagram();
};

const updateDiagram = () => {
  if (diagram) {
    diagram.model = new go.GraphLinksModel(nodes.value, links.value);
  }
};
onMounted(() => {
  initDiagram();
});

watch(nodes, updateDiagram);
watch(links, updateDiagram);
watch(restrictedArea, (val)=>{
        console.log(val,'rect')
},{
  immediate: true
});

</script>

<style scoped>
/* Add your styles here */
</style>
