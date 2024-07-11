<template>
  <input type="file" id="input" name="input" @change="handle" />
  <button type="button" class="small" @click="exports">输出</button>
  <div ref="graphContainer" style="width: 900; height: 615px"></div>
</template>

<script lang="ts" setup>
import { xml2json } from "xml-js";
import { alldata } from "./allData.json";
import { ref, onMounted } from "vue";
import {
  Graph,
  GraphDataModel,
  MaxToolbar,
  Client,
  Cell,
  Geometry,
  CellStyle,
  ImageBox,
  InternalEvent,
  constants,
  gestureUtils,
  styleUtils,
  RubberBandHandler,
  ConnectionHandler,
  Point,
  eventUtils,
  ModelXmlSerializer,
  popup,
  EdgeStyle,
  GraphHierarchyEdge,
  Shape,
  TriangleShape,
  EdgeHandler,
  ConnectionConstraint,
  CellState,
  ConstraintHandler,
  mathUtils,
  utils,
  Rectangle,
  DragSource,
  PanningHandler,
} from "@maxgraph/core";
const inputElement = document.getElementById("input");
const graphContainer = ref<HTMLElement | null>(null);
const graphs = ref<any>();
// 默认节点数据
let dataList = {
  point: [
    {
      id: "M01",
      type: "T-NIU",
      x: 5000,
      y: 9000,
    },
    {
      id: "M02",
      type: "T-NIU",
      x: 2400,
      y: 6000,
    },
    {
      id: "S01",
      type: "I-NIU",
      x: 2600,
      y: 4000,
    },
    {
      id: "S02",
      type: "I-NIU",
      x: 2800,
      y: 1200,
    },
    {
      id: "S03",
      type: "Switch",
      x: 3500,
      y: 2000,
    },
  ],
  to: [
    { from: "M01", to: "S01" },
    { from: "M02", to: "S02" },
    { from: "S01", to: "S02" },
  ],
};

// 限制区域数据
let restrictedAreaCoordinates = [
  { x: 0, y: 0 },
  { x: 0, y: 4000 },
  { x: 15000, y: 17000 },
  { x: 12400, y: 4000 },
  { x: 15000, y: 0 },
];
let groupEdges = ref<any>([]);
let inPoint; //默认初始化的点线
let initPaths = []; //初始化的默认连接线
let exportPath;
let edgePath;
function isPointInPolygon(
  point: { x: number; y: number },
  polygon: { x: number; y: number }[]
): boolean {
  let inside = false;
  for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
    const xi = polygon[i].x;
    const yi = polygon[i].y;
    const xj = polygon[j].x;
    const yj = polygon[j].y;

    const intersect =
      yi > point.y !== yj > point.y &&
      point.x < ((xj - xi) * (point.y - yi)) / (yj - yi) + xi;
    if (intersect) inside = !inside;
  }
  return inside;
}
function isPointInRestrictedArea(point: { x: number; y: number }): boolean {
  // 使用你之前定义的 isPointInPolygon 函数来判断点是否在限制区域内
  return isPointInPolygon(point, restrictedAreaCoordinates);
}
function drawPolygon(graph: Graph, coordinates: any) {
  const model = graph.getDataModel();
  model.beginUpdate();
  try {
    const parent = graph.getDefaultParent();

    // 创建多边形顶点
    const points = [];
    coordinates.forEach((coord) => {
      const vertex = graph.insertVertex(
        parent,
        null,
        "",
        coord.x,
        coord.y,
        0,
        0,
        { shape: "ellipse", movable: false }
      );
      vertex.setConnectable(false); // 防止顶点被连接
      points.push(vertex);
    });

    // 连接多边形顶点
    for (let i = 0; i < points.length; i++) {
      const source = points[i];
      const target = points[(i + 1) % points.length];
      const edge = graph.insertEdge(parent, null, "", source, target);
      edge.setStyle({
        endArrow: "none",
        movable: false,
        edgeStyle: "noEdgeStyle",
        deletable: false,
        strokeColor: "inherit",
      }); // 设置连接线无箭头且不能移动
    }
  } finally {
    model.endUpdate();
  }
}
interface CustomCell extends Cell {
  isAxis?: boolean;
  isType?: string;
}
const initialContainerWidth = 20000; // 原始容器的宽度
const initialContainerHeight = 20000; // 原始容器的高度
onMounted(() => {
  getDataMessage();
});

function getNeighborsNew(graph: Graph, cell) {
  const neighbors = [];
  if (!cell) return neighbors; // 检查节点是否存在
  const edges = cell.edges || [];
  edges.forEach((edge) => {
    if (cell.isType !== "T-NIU" && edge.target.isType !== "I-NIU") {
      // 当前元素不是出口节点 且目标元素不是入口节点
      const target = edge.target;
      neighbors.push(target); // 将边的target添加为邻居
    }
  });
  return neighbors;
}
function getNeighbors(graph: Graph, cell) {
  const neighbors = [];
  if (!cell) return neighbors; // 检查节点是否存在
  const edges = graph.getEdges(cell);
  edges.forEach((edge) => {
    const source = edge.source;
    const target = edge.target;
    // if (source !== cell) {
    //   neighbors.push(source);
    // } else {
    //   neighbors.push(target);
    // }
    if (source === cell) {
      neighbors.push(target);
    } else if (target === cell) {
      neighbors.push(source);
    }
    // if (source === cell) {
    //   neighbors.push(target); // 将边的target添加为邻居
    // }
  });
  return neighbors;
}

function isDefaultNode(nodeId, pointsMap) {
  return Object.values(pointsMap).some(
    (point) =>
      point.id === nodeId &&
      (point.isType === "I-NIU" || point.isType === "T-NIU")
  );
}

function isSubpath(arr, subpath) {
  if (subpath.length >= arr.length) return false;
  for (let i = 0; i <= arr.length - subpath.length; i++) {
    let match = true;
    for (let j = 0; j < subpath.length; j++) {
      if (arr[i + j] !== subpath[j]) {
        match = false;
        break;
      }
    }
    if (match) return true;
  }
  return false;
}

function removeSubpathsNew(paths) {
  const arr = paths.map((item) => {
    return item.join("-");
  });
  // 用于存储需要移除的索引
  let toRemove = new Set();

  // 遍历数组中的每个字符串
  for (let i = 0; i < arr.length; i++) {
    for (let j = 0; j < arr.length; j++) {
      if (i !== j && arr[j].includes(arr[i])) {
        toRemove.add(i);
      }
    }
  }

  // 过滤掉需要移除的字符串
  return arr
    .filter((_, index) => !toRemove.has(index))
    .map((item) => item.split("-"));
}


function removeSubpaths(paths) {
  return paths.filter(
    (path, index) =>
      !paths.some(
        (otherPath, otherIndex) =>
          otherIndex !== index && isSubpath(otherPath, path)
      )
  );
}

function findPaths(
  graph: Graph,
  sourceId,
  targetId,
  visited,
  path,
  paths,
  pointsMap
) {
  visited.add(sourceId);
  path.push(sourceId);
  const cell = graph.getDataModel().getCell(sourceId);
  const neighbors = getNeighbors(graph, cell);
  if (sourceId === targetId) {
    // 当找到完整路径时，无论路径末尾节点是否为默认节点，都记录路径
    paths.push([...path]);
  } else {
    let defaultNodeCount = path.filter((nodeId) =>
      isDefaultNode(nodeId, pointsMap)
    ).length;
    for (const neighbor of neighbors) {
      const neighborId = neighbor.getId();
      const edges = graph.getDataModel().getEdgesBetween(cell, neighbor);
      const edge = edges.length > 0 ? edges[0] : null;
      const isSourceNode = edge && edge.source.getId() === sourceId; // 确保当前节点是边的source
      // 如果邻居节点未访问且（路径中默认节或者邻居节点不是默认节点），则继续DFS
      if (!visited.has(neighborId) && defaultNodeCount < 2 && isSourceNode) {
        findPaths(graph, neighborId, targetId, visited, path, paths, pointsMap);
      }
    }
  }
  path.pop();
  visited.delete(sourceId);
}

// 深度遍历函数
function dfs(filteredArr, start, path = [], paths = [], graph) {
  path.push(start.id);

  // 记录路径
  paths.push([...path]);

  // 递归遍历相邻节点
  const neighbors = getNeighborsNew(graph, start);
  for (const neighbor of neighbors || []) {
    if (!path.includes(neighbor.id)) {
      // 避免环路
      dfs(filteredArr, neighbor, path, paths);
    }
  }

  // 回溯
  path.pop();
  return paths;
}

// 遍历所有节点之间的路径
function getAllPaths(filteredArr, graph) {
  let allPaths = [];
  for (const startNode of filteredArr) {
    const pathsFromNode = dfs(filteredArr, startNode, [], [], graph);
    if (pathsFromNode.length >= 2) {
      allPaths = allPaths.concat(pathsFromNode);
    }
  }
  return allPaths;
}
function findAndRemoveDefaultEdgesNew(graph: Graph, pointsMap) {
  const filteredArr = Object.values(graph.model.cells).filter((item) =>
    item.hasOwnProperty("isType")
  );

  const allPaths = getAllPaths(filteredArr, graph);

  console.log(allPaths, "----allPaths");

  const result = removeSubpathsNew(allPaths);
  console.log("Paths found:", result);
  exportPath = result;
  const grouped = result.reduce((acc, subArray) => {
    const firstElement = subArray[0];
    const lastElement = subArray[subArray.length - 1];
    const key = `${firstElement}-${lastElement}`;

    if (!acc[key]) {
      acc[key] = [];
    }

    acc[key].push(subArray);

    return acc;
  }, {});
  console.log(grouped, "grouped");
  groupEdges.value = grouped;
  deletedEdges(graph, grouped);
}
function findAndRemoveDefaultEdges(graph: Graph, pointsMap) {
  const model = graph.getDataModel();
  const paths = [];
  const filteredArr = Object.values(graph.model.cells).filter((item) =>
    item.hasOwnProperty("isType")
  );
  const initPointKeys = Object.values(pointsMap);
  for (let i = 0; i < filteredArr.length; i++) {
    for (let j = i + 1; j < filteredArr.length; j++) {
      const sourceId = filteredArr[i].id;
      const targetId = filteredArr[j].id;
      const visited = new Set();
      const path = [];
      findPaths(graph, sourceId, targetId, visited, path, paths, pointsMap);
    }
  }
  const result = removeSubpaths(paths);
  console.log("Paths found:", result);
  exportPath = result;
  const grouped = result.reduce((acc, subArray) => {
    const firstElement = subArray[0];
    const lastElement = subArray[subArray.length - 1];
    const key = `${firstElement}-${lastElement}`;

    if (!acc[key]) {
      acc[key] = [];
    }

    acc[key].push(subArray);

    return acc;
  }, {});
  console.log(grouped, "grouped");
  groupEdges.value = grouped;
  deletedEdges(graph, grouped);
}
function deletedEdges(graph: Graph, grouped) {
  const result = Object.values(grouped).map((group) => {
    const model = graph.getDataModel();
    // console.log(group,'grap')
    if (group.length >= 2) {
      const path = group[0];
      const source = path[0];
      const target = path[path.length - 1];
      const edges = graph.getEdgesBetween(
        model.getCell(source),
        model.getCell(target)
      );
      console.log(edges, "edges");
      graph.removeCells(edges);
    }
  });
}
const init = () => {
  if (graphContainer.value) {
    configureImagesBasePath();

    const div = document.createElement("div");
    div.style.display = "flex";
    div.style.width = "100%";
    div.style.height = "100%";

    const tbContainer = document.createElement("div");
    tbContainer.style.display = "flex";
    tbContainer.style.flexDirection = "column";
    tbContainer.style.marginRight = ".5rem";
    div.appendChild(tbContainer);

    const toolbar = new MaxToolbar(tbContainer);
    toolbar.enabled = false;
    const containerWidth = graphContainer.value.clientWidth;
    const containerHeight = graphContainer.value.clientHeight;

    dataList.point.forEach((point) => {
      point.x = (point.x / initialContainerWidth) * containerWidth;
      point.y =
        containerHeight - (point.y / initialContainerHeight) * containerHeight;
    });

    restrictedAreaCoordinates.forEach((point) => {
      point.x = (point.x / initialContainerWidth) * containerWidth;
      point.y =
        containerHeight - (point.y / initialContainerHeight) * containerHeight;
    });
    Shape.prototype.getPorts = function () {
      const numPorts = 8; // 8个端口，可以根据需要调整
      const ports = {};

      for (let i = 0; i < numPorts; i++) {
        const angle = (360 / numPorts) * i;
        const radians = angle * (Math.PI / 180);
        const x = Math.cos(radians) * 0.5 + 0.5; // 0.5是圆心x坐标，这里假设圆半径为0.5
        const y = Math.sin(radians) * 0.5 + 0.5; // 0.5是圆心y坐标，这里假设圆半径为0.5

        ports[`port${i}`] = {
          x: x,
          y: y,
          perimeter: true,
          constraint: `${angle} degrees`,
        };
      }

      return ports;
    };
    class MyCustomGraph extends Graph {
      // Define constraints for connections
      getAllConnectionConstraints(terminal: any) {
        const cell = terminal?.cell as CustomCell;
        if (cell != null && cell.isVertex() && !cell.isAxis) {
          if (terminal.shape != null) {
            const ports = terminal.shape.getPorts();
            const cstrs: ConnectionConstraint[] = [];
            for (const id in ports) {
              const port = ports[id];
              const cstr = new ConnectionConstraint(
                new Point(port.x, port.y),
                port.perimeter
              );
              cstr.id = id;
              cstrs.push(cstr);
            }
            return cstrs;
          }
        }
        return null;
      }
    }
    const container = createGraphContainer();
    div.appendChild(container);
    const model = new GraphDataModel();
    const graph = new MyCustomGraph(container, model);
    graphs.value = graph;
    addMouseWheelZoom(graph, container);
    redrawAxes(graph, currentScale);
    const connectionHandler = graph.getPlugin(
      "ConnectionHandler"
    ) as ConnectionHandler;
    connectionHandler.connectImage = new ImageBox(
      `${Client.imageBasePath}/connector.gif`,
      16,
      16
    );

    //  new RubberBandHandler(graph);
    graph.setPanning(true); 
  const panningHandler = new PanningHandler(graph);

// 配置使用左键进行拖拽
panningHandler.useLeftButtonForPanning = true;
panningHandler.panningEnabled = true
// 将 PanningHandler 应用到图形实例中
graph.setPanning(true); 
    graph.setPortsEnabled(false);
    graph.setConnectable(true);
    graph.setMultigraph(false);
    graph.setCellsResizable(false);
    connectionHandler.isConnectableCell = function (cell: any) {
      return false;
    };
    const graphGetConnectionPoint = graph.getConnectionPoint;
    graph.getConnectionPoint = function (vertex: any, constraint: any) {
      if (constraint.id != null && vertex != null && vertex.shape != null) {
        const port = vertex.shape.getPorts()[constraint.id];
        if (port != null) {
          constraint = new ConnectionConstraint(
            new Point(port.x, port.y),
            port.perimeter
          );
        }
      }
      return graphGetConnectionPoint.apply(this, arguments);
    };
    utils.getPortConstraints = function (terminal, edge, source, defaultValue) {
      let key = source
        ? constants.STYLE_SOURCE_PORT
        : constants.STYLE_TARGET_PORT;
      let id = edge.style[key];
      let port = terminal.shape.getPorts()[id];
      // TODO: Add support for rotation, direction
      if (port != null) {
        return port.constraint;
      }
      return mxUtilsGetPortConstraints.apply(this, arguments);
    };
    // Connect preview
    connectionHandler.createEdgeState = function (me) {
      let edge = graph.createEdge(null, null, null, null, null);
      return new CellState(
        this.graph.view,
        edge,
        this.graph.getCellStyle(edge)
      );
    };
    // addDefaultDataPointsAndConnections(graph, dataList);
    const initpoints = addDefaultDataPointsAndConnections(graph, dataList);
    inPoint = initpoints;
    console.log(initpoints, "initpoints");
    new RubberBandHandler(graph);
    drawPolygon(graph, restrictedAreaCoordinates);
    drawAxes(graph); // 初始化时绘制坐标轴
    // container.addEventListener('wheel', handleMouseWheel);
    // 监听容器大小变化
    window.addEventListener("resize", () => {
      drawAxes(graph);
    });
    // addVertex('images/rectangle.gif', 20, 10, {}, "T-NIU");
    addVertex(
      "images/ellipse.gif",
      20,
      20,
      {
        shape: "ellipse",
        perimeter: "ellipsePerimeter",
        fillColor: "yellow",
      },
      "Switch"
    );
    // addVertex('images/rectangle.gif', 20, 10, {}, "I-NIU");
    addVertex("images/rectangle.gif", 10, 10, { fillColor: "blue" }, "Link");

    function addVertex(
      icon: string,
      w: number,
      h: number,
      style: CellStyle,
      title: string
    ) {
      const vertex = new Cell(
        title,
        new Geometry(0, 0, w, h),
        style
      ) as CustomCell;
      vertex.isType = title;
      vertex.setVertex(true);
      const img: any = addToolbarItem(graph, toolbar, vertex, icon);
      img.enabled = true;
      graph.getSelectionModel().addListener(InternalEvent.CHANGE, () => {
        const tmp = graph.isSelectionEmpty();
        styleUtils.setOpacity(img, tmp ? 100 : 20);
        img.enabled = tmp;
      });
    }

    function addToolbarItem(
      graph: Graph,
      toolbar: MaxToolbar,
      prototype: Cell,
      image: string
    ) {
      const dropHandler = (
        graph: Graph,
        _evt: MouseEvent,
        _cell: Cell | null,
        x?: number,
        y?: number
      ) => {
        graph.stopEditing(false);
        const vertex = graph.getDataModel().cloneCell(prototype)!;
        if (vertex?.geometry) {
          const isInRestrictedArea = isPointInRestrictedArea({
            x: x || 0,
            y: y || 0,
          });
          if (isInRestrictedArea) {
            // 如果在限制区域内，则添加新节点
            x !== undefined && (vertex.geometry.x = x);
            y !== undefined && (vertex.geometry.y = y);
            graph.addCell(vertex, null);
            graph.setSelectionCell(vertex);
          } else {
            // 如果不在限制区域内，则不添加新节点
            console.log("Cannot drop outside the restricted area.");
          }
        }
      };

      const img: any = toolbar.addMode(
        null,
        image,
        (evt: MouseEvent, cell: Cell) => {
          const pt = graph.getPointForEvent(evt);
          dropHandler(graph, evt, cell, pt.x, pt.y);
        },
        ""
      );

      InternalEvent.addListener(img, "mousedown", (evt: MouseEvent) => {
        if (img.enabled == false) {
          InternalEvent.consume(evt);
        }
      });

      gestureUtils.makeDraggable(img, graph, dropHandler);
      return img;
    }

    function addDefaultDataPointsAndConnections(graph: Graph, data: any) {
      const dataPointStyle: CellStyle = {
        shape: "ellipse",
        perimeter: "ellipsePerimeter",
      };
      const edgeStyle: CellStyle = {
        edgeStyle: "noEdgeStyle",
        // edgeStyle: 'elbowEdgeStyle',
        strokeColor: "gray",
        strokeWidth: 1,
        dashed: true,
        movable: false
      };
      const style = graph.getStylesheet().getDefaultEdgeStyle();
      // @ts-ignore TODO fix the type, this works
      style.edgeStyle = EdgeStyle.ElbowConnector;
      style.strokeColor = "#000000";
      const model = graph.getDataModel();
      model.beginUpdate();
      try {
        const parent = graph.getDefaultParent();

        // Add data points
        const pointsMap: { [key: string]: Cell } = {};
        dataList.point.forEach((point) => {
          const vertexWidth = 20;
          const vertexHeight = 10;
          const adjustedX = point.x - vertexWidth / 2;
          const adjustedY = point.y - vertexHeight / 2;
          const iniuStyle: CellStyle = {
            deletable: false,
            fillColor: point.type === "I-NIU" ? "#F8CECC" : "inherit",
          };
          // pointsMap[point.id] = graph.insertVertex(parent, null, point.id, adjustedX, adjustedY, vertexWidth, vertexHeight, iniuStyle);
          const vertex = graph.insertVertex(
            parent,
            null,
            point.id,
            adjustedX,
            adjustedY,
            vertexWidth,
            vertexHeight,
            iniuStyle
          ) as CustomCell;
          vertex.isType = point.type;
          pointsMap[point.id] = vertex;
          pointsMap[point.id].setConnectable(true);
        });

        dataList.to.forEach((toItem) => {
          const sourceCell = pointsMap[toItem.from];
          const targetCell = pointsMap[toItem.to];
          if (sourceCell && targetCell) {
            const edge = graph.insertEdge(
              parent,
              null,
              "",
              sourceCell,
              targetCell,
              edgeStyle
            );
            edge.setConnectable(true);
            const path = [];
            path.push(sourceCell.id, targetCell.id);
            initPaths.push(path);
          }
        });
        return pointsMap;
      } finally {
        model.endUpdate();
      }
    }
    graph.addListener(InternalEvent.CELLS_MOVING, (sender, evt) => {
      const cells = evt.getProperty("cells");
      const dx = evt.getProperty("dx");
      const dy = evt.getProperty("dy");

      cells.forEach((cell: Cell) => {
        if (cell.geometry) {
          const newX = cell.geometry.x + dx;
          const newY = cell.geometry.y + dy;
          const isInRestrictedArea = isPointInPolygon(
            { x: newX, y: newY },
            restrictedAreaCoordinates
          );
          console.log(dx, dy, cell.geometry.x, cell.geometry.y);
          if (!isInRestrictedArea) {
            evt.consume();
          }
        }
      });
    });
    graph.addListener(InternalEvent.CELLS_MOVED, (sender, evt) => {
      const cells = evt.getProperty("cells");
      const dx = evt.getProperty("dx");
      const dy = evt.getProperty("dy");

      cells.forEach((cell: Cell) => {
        if (cell.geometry) {
          const newX = cell.geometry.x + dx;
          const newY = cell.geometry.y + dy;
          const height = cell.geometry.height;
          console.log({ x: newX, y: newY + height });
          const isInRestrictedArea = isPointInRestrictedArea({
            x: newX,
            y: newY,
          });
          const isInRestrictedAreatop = isPointInRestrictedArea({
            x: newX,
            y: newY - height,
          });
          const isInRestrictedAreabottom = isPointInRestrictedArea({
            x: newX,
            y: newY + height,
          });
          if (
            !isInRestrictedArea &&
            !isInRestrictedAreatop &&
            !isInRestrictedAreabottom
          ) {
            cell.geometry.x -= dx;
            cell.geometry.y -= dy;
          }
          // console.log(dx,dy, cell.geometry.x , cell.geometry.y)
        }
      });
    });
    connectionHandler.addListener(InternalEvent.CONNECT, (sender, evt) => {
      const edge = evt.getProperty("cell") as Cell;
      const source = edge.getTerminal(true);
      const target = edge.getTerminal(false);

      if (source && target) {
        // 1.获取初始化默认cell(注意是cell，不是获取的默认json)
        // console.log(initpoints,'initpoints')
        // 2.深度遍历找到与之相关边
        // 3.建成path链表
        // getNeighbors(graph,initpoints)
        // 4.path链表集成数组paths
        // 5.paths中链表头部和尾部有相同删除默认线条
        // findAndRemoveDefaultEdges(graph, initpoints);
        findAndRemoveDefaultEdgesNew(graph, initpoints);
      }
    });
    // 启用顶点标签编辑和唯一名称验证
    graph.setCellsEditable(true);
    graph.addListener(InternalEvent.CHANGE, (sender, evt) => {
      console.log("CHANGE event triggered");
      const changes = evt.getProperty("edit").changes;
      for (const change of changes) {
        if (change.constructor.name === "mxValueChange") {
          const cell = change.cell as Cell;
          if (cell.isVertex()) {
            const newValue = change.value as string;
            if (!isUniqueVertexName(newValue, graph)) {
              alert("顶点名称必须唯一。");
              cell.setValue(change.previous);
            }
          }
        }
      }
    });
    graph.addListener(InternalEvent.LABEL_CHANGED, (sender, evt) => {
      const cell = evt.getProperty("cell") as Cell;
      const newValue = evt.getProperty("value") as string;
      const oldValue = evt.getProperty("old");
      console.log(newValue, "new");
      if (!isUniqueVertexName(newValue, graph)) {
        alert("节点名称不能重复，请重新命名。");
        console.log(oldValue, "cell.getValue()");
        // evt.properties.value = evt.properties.old
        cell.setValue(oldValue);
        evt.consume();
      }
    });
    // 监听双击事件以启动标签编辑
    graph.addListener(InternalEvent.DOUBLE_CLICK, (sender, evt) => {
      const cell = evt.getProperty("cell") as Cell;
      if (cell && cell.isVertex()) {
        graph.startEditingAtCell(cell);
      }
    });
    graphContainer.value.appendChild(div);
  }
};
function isUniqueVertexName(name: string, graph: Graph): boolean {
  const allCells = graph.getDataModel().cells;
  let index = 0;
  for (const id in allCells) {
    if (allCells.hasOwnProperty(id)) {
      const cell = allCells[id] as Cell;
      if (cell.isVertex() && cell.getValue() === name) {
        index++;
        if (index >= 2) {
          return false;
        }
      }
    }
  }
  return true;
}
function createGraphContainer(): HTMLDivElement {
  const container = document.createElement("div");
  container.style.width = "100%";
  container.style.height = "100%";
  container.style.border = "1px solid black";
  container.style.position = "relative";
  container.style.overflow = "hidden";
  return container;
}

function configureImagesBasePath() {
  Client.imageBasePath = "/images";
}
function drawAxes(graph: Graph) {
  const container = graphContainer.value;
  if (!container || !graph) return;

  const containerWidth = container.clientWidth - 34;
  const containerHeight = container.clientHeight;

  const model = graph.getDataModel();
  model.beginUpdate();
  try {
    const parent = graph.getDefaultParent();

    // X轴
    const xAxisLength = 20000 * currentScale; // X轴的总长度
    const xAxisStep = 1000 * currentScale; // X轴刻度间隔
    for (let i = 0; i <= xAxisLength; i += xAxisStep) {
      const label = i.toString();
      const x = (i / xAxisLength) * containerWidth;
      const vertex = graph.insertVertex(
        parent,
        null,
        label,
        x,
        containerHeight - 10,
        1,
        10,
        {
          strokeColor: "#000000",
          strokeWidth: 1,
          fillColor: "#000000",
          movable: false,
        }
      ) as CustomCell;
      vertex.isAxis = true;
    }

    // Y轴
    const yAxisLength = 20000 * currentScale; // Y轴的总长度
    const yAxisStep = 1000 * currentScale; // Y轴刻度间隔
    for (let i = 0; i <= yAxisLength; i += yAxisStep) {
      const label = i.toString();
      const y = (i / yAxisLength) * containerHeight;
      const vertex = graph.insertVertex(
        parent,
        null,
        label,
        10,
        containerHeight - y,
        10,
        1,
        {
          strokeColor: "#000000",
          strokeWidth: 1,
          fillColor: "#000000",
          movable: false,
        }
      ) as CustomCell;
      vertex.isAxis = true;
    }
  } finally {
    model.endUpdate();
  }
}
function convertToOriginalCoordinates(
  point: { x: number; y: number },
  containerWidth: number,
  containerHeight: number
): { x: number; y: number } {
  const originalX = (point.x / containerWidth) * initialContainerWidth;
  const originalY =
    initialContainerHeight -
    (point.y / containerHeight) * initialContainerHeight;
  return { x: originalX, y: originalY };
}
const exports = () => {
  const model = graphs.value.getDataModel();
  const parent = graphs.value.getDefaultParent();
  const cells = graphs.value.getChildVertices(parent);
  // console.log(cells,'cells')
  const container = graphContainer.value;
  const containerWidth = container
    ? container.clientWidth
    : initialContainerWidth;
  const containerHeight = container
    ? container.clientHeight
    : initialContainerHeight;
  console.log(exportPath,inPoint,initPaths, "exportPath");
//   const result = exportPath.filter(a1 => !initPaths.some(a2 => JSON.stringify(a1) === JSON.stringify(a2)));
// console.log(result,'filter')
// const ids = Object.values(inPoint).map(point => point.id);

// const uniqueArray = [...new Set(result.flat())];
// const missingElements = uniqueArray.filter(element => !ids.includes(element));
// console.log(missingElements,'miss')
// const getNode = missingElements.map(nodeId=>{
//  return  {
//         object: {
//           $: { disabled: "0", kind: "dtpLink", name: `${model.getCell(nodeId).value}` },
//           properties: [
//             {
//               entry: [
//                 {
//                   $: { key: "common" },
//                   entry: [
//                     { $: { key: "clock", value: "(switchBasedArchitecture:ACLKRegime/Cm/root)" } }
//                   ]
//                 },
//                 {
//                   $: { key: "datapath" },
//                   entry: [
//                     {
//                       $: { key: "serialization" },
//                       entry: [
//                         { $: { key: "headerPenalty", value: "NONE" } },
//                         { $: { key: "nBytePerWord", value: "16" } }
//                       ]
//                     }
//                   ]
//                 }
//               ]
//             }
//           ]
//         }
//       };
// })
// this.xmlData.root.objects.push(newObject);
// console.log(getNode)

  // cells.forEach(cell => {
  //   if (cell.geometry) {
  //     const originalCoords = convertToOriginalCoordinates({ x: cell.geometry.x, y: cell.geometry.y }, containerWidth, containerHeight);
  //     cell.geometry.x = originalCoords.x;
  //     cell.geometry.y = originalCoords.y;
  //   }
  // });
  const vertexWidth = 20;
  const vertexHeight = 10;

  cells.forEach((cell) => {
    if (cell.geometry) {
      const adjustedX = cell.geometry.x + vertexWidth / 2;
      const adjustedY = cell.geometry.y + vertexHeight / 2;
      const originalCoords = convertToOriginalCoordinates(
        { x: adjustedX, y: adjustedY },
        containerWidth,
        containerHeight
      );
      cell.geometry.x = originalCoords.x;
      cell.geometry.y = originalCoords.y;
    }
  });
  const xml = new ModelXmlSerializer(model).export();
//   const jsonResult = xml2json(xml, { compact: true, spaces: 4 });
//   const conectData  = JSON.parse(jsonResult)
// console.log(conectData,'conectData')
// const data = conectData.object.object.filter(item=>)
  // Convert back to view coordinates if necessary
  cells.forEach((cell) => {
    if (cell.geometry) {
      const viewCoords = {
        x: (cell.geometry.x / initialContainerWidth) * containerWidth,
        y:
          containerHeight -
          (cell.geometry.y / initialContainerHeight) * containerHeight,
      };
      cell.geometry.x = viewCoords.x;
      cell.geometry.y = viewCoords.y;
    }
  });
};
const getDataMessage = () => {
  // console.log(alldata,'all')
  restrictedAreaCoordinates = [];
  dataList.point = [];
  alldata.map((item) => {
    if (item.type === "vertex") {
      const items = {
        ...item,
        x: item["x-pos"],
        y: item["y-pos"],
      };
      restrictedAreaCoordinates.push(items);
    } else {
      const items2 = {
        id: item.name,
        ...item,
        x: item["x-pos"],
        y: item["y-pos"],
      };
      dataList.point.push(items2);
    }
  });
};
inputElement?.addEventListener("change", (event) => {
  console.log(event, "event");
});
let currentScale = 1;
function addMouseWheelZoom(graph: Graph, container: HTMLDivElement) {
  let scaleFactor = 1.1;
  let currentScale = 1;

  container.addEventListener("wheel", (evt: WheelEvent) => {
    evt.preventDefault();
    const rect = container.getBoundingClientRect();
    const left = rect.left;
    const bottom = rect.bottom;
    const prevScale = currentScale;

    if (evt.deltaY > 0) {
      if (currentScale > 1) {
        currentScale /= scaleFactor;
      } else {
        return;
      }
    } else {
      // Zoom in
      currentScale *= scaleFactor;
    }
    // console.log(prevScale,currentScale)
    // Calculate new translate values to keep the left bottom corner fixed
    const newTranslateX = left / prevScale - left / currentScale;
    const newTranslateY = bottom / prevScale - bottom / currentScale;
    // console.log( graph.view.translate.y,newTranslateY, graph.view.translate.y - newTranslateY,'newY')
    graph.view.scaleAndTranslate(
      currentScale,
      0,
      evt.deltaY < 0
        ? graph.view.translate.y - newTranslateY + 5
        : graph.view.translate.y - newTranslateY - 5
    );
  });
}
// function addMouseWheelZoom(graph: Graph, container: HTMLDivElement) {
//   let scaleFactor = 1.1;
//   let currentScale = 1;

//   container.addEventListener('wheel', (evt: WheelEvent) => {
//     evt.preventDefault();

//     const prevScale = currentScale;

//     if (evt.deltaY > 0) {
//       // Zoom out
//       currentScale /= scaleFactor;
//     } else {
//       // Zoom in
//       currentScale *= scaleFactor;
//     }

//     // Calculate new translate values to keep the left bottom corner fixed
//     const newTranslateX = 0;
//     const newTranslateY = container.clientHeight * (1 - currentScale / prevScale);

//     graph.view.scaleAndTranslate(
//       currentScale,
//       newTranslateX,
//       graph.view.translate.y + newTranslateY
//     );
//   });
// }
// function addMouseDown(graph: Graph, container: HTMLDivElement){
//   container.addEventListener('mousedown', (evt) => {
//     evt.preventDefault();
//     console.log(evt)
//   })
// }

function redrawAxes(graph: Graph, scale: number) {
  const container = graphContainer.value;
  if (!container) return;

  const containerWidth = container.clientWidth - 34;
  const containerHeight = container.clientHeight;

  const model = graph.getDataModel();
  model.beginUpdate();
  try {
    const parent = graph.getDefaultParent();

    // Clear existing axis
    const children = parent.getChildren();
    for (let i = children.length - 1; i >= 0; i--) {
      const cell = children[i];
      if (cell.isAxis) {
        graph.removeCells([cell]);
      }
    }

    // Redraw X-axis
    const xAxisLength = 20000;
    const xAxisStep = 1000 * scale;
    for (let i = 0; i <= xAxisLength; i += xAxisStep) {
      const label = (i / scale).toFixed(0);
      const x = (i / xAxisLength) * containerWidth;
      const vertex = graph.insertVertex(
        parent,
        null,
        label,
        x,
        containerHeight - 10,
        1,
        10,
        {
          strokeColor: "#000000",
          strokeWidth: 1,
          fillColor: "#000000",
          movable: false,
        }
      ) as CustomCell;
      vertex.isAxis = true;
    }

    // Redraw Y-axis
    const yAxisLength = 20000;
    const yAxisStep = 1000 * scale;
    for (let i = 0; i <= yAxisLength; i += yAxisStep) {
      const label = (i / scale).toFixed(0);
      const y = (i / yAxisLength) * containerHeight;
      const vertex = graph.insertVertex(
        parent,
        null,
        label,
        10,
        containerHeight - y,
        10,
        1,
        {
          strokeColor: "#000000",
          strokeWidth: 1,
          fillColor: "#000000",
          movable: false,
        }
      ) as CustomCell;
      vertex.isAxis = true;
    }
  } finally {
    model.endUpdate();
  }
}

const handle = (e) => {
  console.log(e);
  let file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = function (e) {
    const xmlText = e.target.result;
    try {
      const jsonResult = xml2json(xmlText, { compact: true, spaces: 4 });
      console.log(JSON.parse(jsonResult));
      const conectData = JSON.parse(jsonResult);
      console.log(conectData,'conectData')
      const data = conectData.object.object[0].properties.entry;
      let match;
      let matchs;
      let toList = [];
      data.forEach((items: any) => {
        if (items._attributes["key"] === "connectivity") {
          // console.log(items.entry,'entry')
          items.entry.forEach((item: any) => {
            // item._attributes["key"]
            const regex = /specification:([A-Z]\d)\/I\/0/g;
            match = regex.exec(item._attributes["key"]);
            //  console.log(match[1],'title')
            item.entry.forEach((its: any) => {
              if (its._attributes["value"] === "True") {
                const regex = /specification:([A-Z]\d)\/T\/0/g;
                matchs = regex.exec(its._attributes["key"]);
                // console.log(matchs,'m')
                toList.push({ from: match[1], to: matchs[1] });
              }
            });
          });
          console.log(toList, "toList");
          dataList.to = toList;
          init();
        }
      });
      console.log(data, "data");
    } catch (e) {
      console.error("Error converting XML to JSON:", e);
    }
  };
  reader.readAsText(file);
};
document.onkeydown = function (e) {
  // console.log('e',e.code,e.key);
  if (e.key === "Backspace" || e.key === "Delete") {
    e.preventDefault(); // 阻止默认的删除行为
    const selectionCell = graphs.value.getSelectionCell();
    console.log(selectionCell, "selectionCell");

    if (selectionCell && (selectionCell.isEdge() || selectionCell.isVertex())) {
      graphs.value.getDataModel().beginUpdate();
      try {
        if (
          selectionCell.style.deletable === false ||
          selectionCell.style.dashed === true
        )
          return;
        graphs.value.getDataModel().remove(selectionCell);
        reset(graphs.value, inPoint);
      } finally {
        graphs.value.getDataModel().endUpdate();
      }
    }
  }
};
function reset(graph, inPoint) {
  // findAndRemoveDefaultEdges(graph, inPoint);
  findAndRemoveDefaultEdgesNew(graph, inPoint);
  let pathB = [];
  Object.values(groupEdges.value).map((path) => {
    pathB.push(...path);
  });
  console.log(pathB, "groupEdges");

  const arraysEqual = (arr1, arr2) => {
    // if (arr1.length !== arr2.length) return false;
    // for (let i = 0; i < arr1.length; i++) {
    //     if (arr1[i] !== arr2[i]) return false;
    // }
    // return true;
    return (
      arr1[0] === arr2[0] && arr1[arr1.length - 1] === arr2[arr2.length - 1]
    );
  };
  // 找出 b 中在 a 中不存在的子数组
  const missingInA = initPaths.filter(
    (subArrayB) => !pathB.some((subArrayA) => arraysEqual(subArrayA, subArrayB))
  );
  // console.log(missingInA,'miss')
  missingInA.forEach((path) => {
    const edgeStyle: CellStyle = {
      edgeStyle: "noEdgeStyle",
      // edgeStyle: 'elbowEdgeStyle',
      strokeColor: "gray",
      strokeWidth: 1,
      dashed: true,
    };
    const parent = graph.getDefaultParent();
    const sourceCell = findById(inPoint, path[0]);
    const targetCell = findById(inPoint, path[1]);
    console.log(path, "path");
    const edge = graph.insertEdge(
      parent,
      null,
      "",
      sourceCell,
      targetCell,
      edgeStyle
    );
    graph.addCells([edge]);
  });
}
const findById = (obj, targetId) => {
  for (let key in obj) {
    if (obj[key].id === targetId) {
      return obj[key];
    }
  }
  return null; // 如果找不到匹配的 id，返回 null
};
</script>

<style scoped>
div {
  border: 1px solid #000;
}
::v-deep .mxCellEditor {
  position: absolute !important;
}
</style>
