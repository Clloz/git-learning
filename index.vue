<template>
  <div class="wrapper" :class="{ grab: cursorState === 'grab' }" ref="mapWrapper">
    <!--    <div>{{mapInfo.address}}</div>-->
    <canvas v-if="mapInfo.address" class="map-canvas" width="300" height="300" ref="mapCanvas"></canvas>
    <div class="tools">
      <div v-if="mapInfo.address" @click="spinCanvas" class="spin-canvas">
        <a-icon type="redo" />
      </div>
      <div v-if="funcSwitch.navigate" class="show-task-path" @click="toggleTaskPathShow">
        <a-icon v-if="!showTaskPaths" type="eye" />
        <a-icon v-if="showTaskPaths" type="eye-invisible" />
      </div>
    </div>
    <div v-if="funcSwitch.navigate && showTaskPaths" class="task-paths">
      <div>当前任务路径: {{ currentTaskIds }}</div>
      <div>已完成任务路径: {{ doneTaskIds }}</div>
    </div>
    <div v-if="blinkText" class="one-key">{{ blinkText }}</div>
    <div
      class="legends"
      :class="{ 'show-legends': showLegends }"
      @mouseover="showLegends = true"
      @mouseleave="showLegends = false"
    >
      <div class="title">图例</div>
      <div class="legend" v-for="legend in legends" :key="legend.id">
        <div class="color" :style="{ background: legend.color }"></div>
        <div>{{ legend.name }}</div>
      </div>
      <div class="title">机器人实时状态</div>
      <div class="car-realtime-info" v-for="info in carRealtimeInfoConf" :key="info.id">
        <div class="name">{{ info.name }}</div>
        <div class="value">{{ carRealtimeInfo[info.prop] || '--' }}</div>
      </div>
      <div class="slider">
        <a-icon v-if="!showLegends" type="double-right"></a-icon>
        <a-icon v-else type="double-left"></a-icon>
      </div>
    </div>
    <div class="context-menu" v-show="contextmenuList.length" :class="{ active: showContextMenu }" ref="contextMenu">
      <div class="item" v-for="item in contextmenuList" :key="item.id" @click="item.handleClick">{{ item.name }}</div>
    </div>
  </div>
</template>

<script>
import { Vue, Component, Ref, Watch, Prop } from 'vue-property-decorator';
import Store, { mapStates } from '@/components/robot-map/store';
import { throttle } from 'lodash';
import {
  getComputedStyle,
  getDragDirection,
  listToIdKeyObj,
  pointsFormatBatch,
  antiPointsFormatBatch,
  routeArrToObj,
  isInCircle,
  isInSegment,
  isInRectSide,
  rectFormatBatch,
  routeObjtoArr,
} from '@/components/robot-map/util';
// TODO: 需要优化 canvas 渲染次数，尽量减少冗余的重复渲染
@Component({
  // 所有 store 中的属性都用计算属性来获取
  computed: {
    ...mapStates({
      points: 'points', // 当前渲染的所有点位
      segments: 'segments', // 当前渲染的所有路径
      rects: 'rects', // 当前渲染的所有矩形
      car: 'car', // 小车的渲染信息
      taskPath: 'taskPath', // 当前任务路径
      donePath: 'donePath', // 当前任务已完成路径
      selectedPoints: 'selectedPoints', // 当前选中的点位
      selectedPaths: 'selectedPaths', // 当前选中的路径
      hoverPoint: 'hoverPoint', // 当前 hover 的点
      hoverPath: 'hoverPath', // 当前 hover 的路径
      hoverRect: 'hoverRect', // 当前 hover 的矩形
      newSegments: 'newSegments', // 用户单次连续新连接的路径 在巡检地图维护页面单击开始连线，可以连续连线；双击后结束连续连线，需要重新选择起点
      newSegmentsCache: 'newSegmentsCache', // 保存当前用户连接的所有未提交路径
      dashedSegment: 'dashedSegment', // 鼠标和上一个路径点的虚线
      pointCloud: 'pointCloud', // 点云数据
      actionCache: 'actionCache', // 操作缓存 TODO: 用来实现操作撤销，当前尚未实现
    }),
  },
})
export default class BaseRobotMap extends Vue {
  @Ref() mapWrapper; // 最外层的 wrapper 元素
  @Ref() mapCanvas; // canvas 元素
  @Ref() contextMenu; // 右键菜单元素
  // 右键菜单
  get contextmenuList() {
    let contextmenus = [
      {
        id: 0,
        name: '开始删除',
        handleClick: this.handleStartDel,
        condition: this.funcSwitch.route && !this.delMode,
      },
      {
        id: 1,
        name: '确认删除',
        handleClick: this.handleDelConfirm,
        condition: this.funcSwitch.route && this.delMode,
      },
      {
        id: 2,
        name: '取消删除',
        handleClick: this.handleDelCancel,
        condition: this.funcSwitch.route && this.delMode,
      },
      {
        id: 3,
        name: '一键导航',
        handleClick: this.handleNavigate,
        condition: this.funcSwitch.navigate && this.showNavigateMenu && !this.relocationMode,
      },
      {
        id: 4,
        name: '开始重定位',
        handleClick: this.handleRelocation,
        condition:
          this.funcSwitch.navigate &&
          !this.relocationMode &&
          this.isCarLoaded &&
          this.carPosition &&
          this.carPosition.vehicleCoordinate,
      },
      {
        id: 5,
        name: '提交定位',
        handleClick: this.submitRelocation,
        condition: this.funcSwitch.navigate && this.relocationMode,
      },
      {
        id: 10,
        name: '取消重定位',
        handleClick: this.cancelRelocation,
        condition: this.funcSwitch.navigate && this.relocationMode,
      },
      {
        id: 6,
        name: '开始框选',
        handleClick: this.handleRectSelectStart,
        condition: this.funcSwitch.rectSelect && !this.rectSelectMode,
      },
      {
        id: 7,
        name: '结束框选',
        handleClick: this.handleRectSelectEnd,
        condition: this.funcSwitch.rectSelect && this.rectSelectMode,
      },
      {
        id: 8,
        name: '添加区域',
        handleClick: this.handleAddRegionStart,
        condition: this.funcSwitch.addRegion && !this.addRegionMode,
      },
      {
        id: 9,
        name: '结束添加',
        handleClick: this.handleAddRegionEnd,
        condition: this.funcSwitch.addRegion && this.addRegionMode,
      },
      {
        id: 10,
        name: '添加点位',
        handleClick: this.handleAddPointStart,
        condition: this.funcSwitch.route && !this.addPointMode,
      },
      {
        id: 11,
        name: '提交点位',
        handleClick: this.handleAddPointEnd,
        condition: this.funcSwitch.route && this.addPointMode,
      }
    ];

    return contextmenus.filter(contextmenu => contextmenu.condition);
  }

  @Prop(Object) funcSwitch; // 组件功能开关
  // @Watch('funcSwitch', {deep: true}) // 测试功能切换
  // handleFuncSwitchChange() {
  //   this.initComponent()
  // }

  // 地图信息的监听，主要处理地图切换
  @Prop(Object) mapInfo; // 地图信息，包括 imgUrl, originX, originY, proportion
  @Watch('mapInfo', { deep: true })
  handleMapInfoChange() {
    // this.loadImage();
    this.initComponent();
  }

  // 导航点数据的监听，变化时需要重新转换格式并根据当前缩放比例计算
  @Prop({ default: () => [] }) allPoints;
  @Watch('allPoints')
  handleAllPointsChange() {
    // 导航点发生变化时重新计算点位
    this.pointsDataInitFormat();
    // this.reCalcPointsCoordinates()
  }

  // 导航点路径的监听
  @Prop({ default: () => [] }) allRoutes; // 从 server 请求来的当前所有路径连线
  @Watch('allRoutes')
  handleAllRoutesChange() {
    this.store.commit('setData', { name: 'segments', data: routeArrToObj(this.allRoutes) });
  }

  // 任务模板页面的任务查看需要显示的点位
  @Prop({ default: () => [] }) taskPointIds;

  // 检修区域变化的监听
  @Prop({ default: () => [] }) allRegions; // 从 server 请求来的所有已添加的检修区域，对应 canvas 上的矩形
  @Watch('allRegions')
  handleAllRegionsChange() {
    this.reCalcRegionsCoordinates();
  }
  @Prop({ default: 1 }) regionType; // 当前绘制的检修区域类型，默认为 1 ：检修区域，颜色为蓝色；挂牌禁行的 type 为 2，颜色为粉色，提供了一个 setRegionType 的方法来修改检修区域类型

  // 当前任务对应导航点和路径的监听
  @Prop({ default: () => [] }) currentTaskIds; // 从 server 请求来当前任务的点位数组
  @Watch('currentTaskIds')
  handleCurrentTaskIdsChange() {
    this.store.commit('setData', { name: 'taskPath', data: this.currentTaskIds });
  }

  // 当前任务已完成的点和路径的监听
  @Prop({ default: () => [] }) doneTaskIds; // 已经完成的任务点位数组
  @Watch('doneTaskIds', { immediate: true })
  handleDoneTaskIdsChange() {
    this.store.commit('setData', { name: 'donePath', data: this.doneTaskIds });
  }

  // 背景图相关属性
  img = null; // 地图对应的 image 对象，会在 created 中加载
  imageNaturalWidth = 0; // 图片的原始宽高，在缩放计算时使用
  imageNaturalHeight = 0;

  // 缩放相关属性
  scaleRatio = 1; // canvas 的缩放比例
  maxScaleRatio = 5; // 最大缩放比例
  minScaleRatio = 0.2; // 最小缩放比例
  scaleRatioChangeStep = 0.06; // 单次缩放比例
  scaleCenter = { x: 0, y: 0 }; // 缩放中心
  isDrawingRect = false; // 是否在绘制矩形，绘制矩形时禁止缩放

  // 小车相关属性
  carRealtimeInfoConf = [
    {id: 'speed', name: '速度(m/s) :', prop: 'speed'},
    {id: 'battery', name: '电量(%) :', prop: 'battery'},
    {id: 'action', name: '运行动作 :', prop: 'action'},
    {id: 'energy_status', name: '充电状态 :', prop: 'energy_status'},
  ]
  @Prop({default: () => ({})}) carRealtimeInfo; // 机器人的实时数据展示
  @Prop() carInfo; // 小车的图片大小基础信息
  @Prop() carPosition; // 小车的定位信息
  @Watch('carPosition')
  onCarPositionChange() {
    if (this.relocationMode) return;
    this.carPositionTransform();
  }
  // 小车到达导航点之后会有一个边框闪烁三秒
  @Prop({ default: false }) isCarArriving;
  @Watch('isCarArriving')
  onCarArrivingChange(newVal) {
    this.store.commit('toggleCarArriving', newVal);
  }
  carImage = null;
  isHoveringCar = false; // 当前鼠标是否 hover 小车
  relocationMode = false; // 是否在重定位状态
  isCarLoaded = false; // 小车图片是否加载完成
  carRotateStep = 2; // 小车旋转的最小角度

  // 点云相关属性
  @Prop({ default: () => [] }) originPointCloud;
  @Watch('originPointCloud')
  onOriginPointCloudChange() {
    if (this.relocationMode) return;
    this.pointCloudTransform();
  }
  @Watch('car', { deep: true })
  onCarStoreChange() {
    if (this.relocationMode) {
      this.pointCloudTransform();
    }
  }

  // canvas 未缩放点位数据
  unScaledPoints = []; // 初始数据转换后的未经过 scaleRatio 缩放的数据

  // 一键导航相关
  @Prop({ default: '' }) blinkText;
  navigateId = -1; // 当前右键一键导航的点
  @Watch('navigateId')
  onNavigateIdChange() {
    !this.showContextMenu && this.handleReRender(); // 由于 navigateId 没有放到 store 中，所以关闭右键菜单时需要手动触发一次渲染
  }
  get showNavigateMenu() {
    // 当前是否有选中的导航点，根据该项来确定是否显示一键导航菜单
    return this.navigateId !== -1;
  }

  // 缩放后的点位 hash
  get scaledPointsHash() {
    return listToIdKeyObj(this.points, 'id');
  }

  // canvas 绘制相关
  get defaultStrokeStyle() {
    // 使用计算属性，否则无法获取对象 prop 中的属性，该属性设置默认的描边颜色
    return this.funcSwitch.navigate ? 'lightblue' : 'blue'; // 机器人管理和点位部署页面默认描边色为 lightblue，其他页面为 blue
  }
  get defaultFillStyle() {
    return this.funcSwitch.navigate ? 'lightblue' : 'blue'; // 机器人管理和点位部署页面默认填充色为 lightblue，其他页面为 blue
  }

  // defaultTaskColor = '#4949e6'; // 默认的当前任务颜色
  // defaultDoneColor = 'pink'; // 默认的当前任务已完成部分点线的颜色
  // defaultHoverColor = 'green'; // 默认的 hover 高亮颜色
  // defaultSelectedColor = 'red'; // 默认的选中颜色
  // defaultNewPathColor = 'violet'; // 默认的新添加路径颜色
  // defaultRepeatColor = '#0000c3';
  // defaultRepeatDoneColor = '#fb516e' // 默认的重复完成路径（任务路径中会有重复规划的路径）
  // defaultPointRadius = 5; // 默认的点半径
  // defaultLineWidth = 3; // 默认的线宽
  // 用 Props 替代属性，颜色和宽度都设为可配置
  @Prop({ default: '#4949e6' }) defaultTaskColor;
  @Prop({ default: 'pink' }) defaultDoneColor;
  @Prop({ default: '#faad14' }) defaultTaskTemplateColor;
  @Prop({ default: 'green' }) defaultHoverColor;
  @Prop({ default: 'red' }) defaultSelectedColor;
  @Prop({ default: 'violet' }) defaultNewPathColor;
  @Prop({ default: '#0000c3' }) defaultRepeatColor;
  @Prop({ default: '#fb516e' }) defaultRepeatDoneColor;
  @Prop({ default: '#faad14' }) defaultHistoryRegionColor;
  @Prop({ default: 5 }) defaultPointRadius;
  @Prop({ default: 3 }) defaultLineWidth;

  showLegends = false; // 是否隐藏图例
  // 颜色图例，地图组件中颜色较多，添加图例帮助理解
  legends = [
    {
      id: 0,
      color: '#4949e6',
      name: '任务路径/检修区域',
    },
    {
      id: 1,
      color: 'pink',
      name: '已完成路径/挂牌禁行',
    },
    {
      id: 2,
      color: '#0000c3',
      name: '重复路径',
    },
    {
      id: 3,
      color: '#fb516e',
      name: '重复已完成路径',
    },
    {
      id: 4,
      color: 'green',
      name: 'hover高亮',
    },
    {
      id: 5,
      color: 'red',
      name: '选中路径',
    },
    {
      id: 6,
      color: 'violet',
      name: '新添加路径',
    },
    {
      id: 7,
      color: '#faad14',
      name: '禁用检修区域',
    },
  ];

  // 右键菜单及删除
  showContextMenu = false;

  // 页面内
  delMode = false; // 巡检地图维护删除模式, false 时表示为连线状态
  rectSelectMode = false; // 地图选点的框选开关，默认的 mousedown mousemove 处理的是页面的拖动
  addRegionMode = false; // 添加检修区域开关，默认的 mousedown mousemove 处理的是页面的拖动
  addPointMode = false; // 地图维护页面添加导航点

  cursorState = ''; // 鼠标状态，拖动 canvas 时为 grab
  canvasRotateDeg = 0; // canvas 当前的旋转角度
  // handleResize = null;

  showTaskPaths = false; // navigate 为 true 的页面需要显示任务路径和已完成任务路径

  store = new Store(); // 实例化数据处理模块
  @Watch('store', { deep: true })
  onStoreChange() {
    this.mapCanvas && this.handleReRender();
  }

  /**
   * @Description: 旋转 canvas 90°
   */
  spinCanvas() {
    this.canvasRotateDeg += 90;
    this.mapCanvas.style.transform = `rotateZ(${this.canvasRotateDeg}deg)`;
  }

  /**
   * @Description: 点击显示/隐藏当前任务路径
   */
  toggleTaskPathShow() {
    this.showTaskPaths = !this.showTaskPaths;
  }

  /**
   * @Description: 初始化 canvas 的绘制配置，画笔粗细，颜色配置等
   * @Return:
   */
  initCanvasConf() {
    const ctx = this.mapCanvas.getContext('2d');
    ctx.strokeStyle = this.defaultStrokeStyle;
    ctx.lineWidth = this.defaultLineWidth;
    ctx.fillStyle = this.defaultFillStyle;
    ctx.save();
  }

  /**
   * @Description: 给父组件获取信息的接口，比如获取点位和路径等
   */
  getMapData(name) {
    if (name === 'segments') return routeObjtoArr(this.segments);
    if (name === 'allPoints') return this.points;
    if (name === 'points')
      return this.selectedPoints.map(point => {
        return point;
      });
  }

  /**
   * @Description: 绘制页面上的点
   * @Param [{x: Number, y: Number}] point: 绘制点的坐标
   * @Param [Number] radius: 绘制点的半径
   * @Param [String] color: 绘制点的颜色
   */
  drawPoint({ x, y }, color = this.defaultFillStyle, radius = this.defaultPointRadius) {
    const ctx = this.mapCanvas.getContext('2d');
    ctx.beginPath();
    ctx.arc(Math.floor(x), Math.floor(y), radius, 0, Math.PI * 2, true);
    ctx.fillStyle = color;
    ctx.fill();
    ctx.closePath();
    ctx.restore();
  }

  drawPointsBatch(points) {
    points.forEach(point => {
      let color = this.defaultFillStyle;
      if (this.funcSwitch.navigate && this.currentTaskIds.includes(point.id)) color = this.defaultTaskColor;
      if (this.funcSwitch.navigate && this.doneTaskIds.includes(point.id)) color = this.defaultDoneColor;
      if (this.taskPointIds.includes(point.id)) color = this.defaultTaskTemplateColor;
      if (this.funcSwitch.navigate && this.navigateId === point.id) color = this.defaultHoverColor;
      if (this.selectedPoints.includes(point.id)) color = this.defaultSelectedColor;
      if (this.hoverPoint === point.id) color = this.defaultHoverColor;
      this.drawPoint({ x: point.x, y: point.y }, color);

      if (this.hoverPoint === point.id || this.navigateId === point.id)
        this.drawText({ x: point.x, y: point.y }, point.id);
    });
  }

  drawPointCloudBatch(points) {
    points.forEach(point => {
      // 随机颜色
      // const r = Math.random() * 255 | 0;
      // const g = Math.random() * 255 | 0;
      // const b = Math.random() * 255 | 0;
      this.drawPoint({ x: point.x, y: point.y }, `#52c41a`, 1);
      // this.drawPoint({x: point.x, y: point.y}, `black`, 1);
    });
  }

  /**
   * @Description: 绘制页面上的线
   * @Param [{x: Number, y: Number}] p1: 线的端点
   * @Param [{x: Number, y: Number}] p2: 线的端点
   * @Param [Number] lineWidth: 绘制的线的宽度
   * @Param [String] color: 绘制线的颜色
   */
  drawSegment(p1, p2, color = this.defaultStrokeStyle, lineWidth = this.defaultLineWidth) {
    const ctx = this.mapCanvas.getContext('2d');
    ctx.strokeStyle = color;
    ctx.lineWidth = lineWidth;
    ctx.beginPath();
    ctx.setLineDash([]);
    ctx.moveTo(Math.floor(p1.x) + 0.5, Math.floor(p1.y) + 0.5);
    ctx.lineTo(Math.floor(p2.x) + 0.5, Math.floor(p2.y) + 0.5);
    ctx.stroke();
    ctx.closePath();
  }

  /**
   * @Description: 判断是否任务路径
   * @Param:
   * @Return:
   */
  checkTaskPath(startId, endId, isDone) {
    const routes = isDone ? this.doneTaskIds : this.currentTaskIds;
    let pathTimes = 0; // 记录是否有重复路径
    for (let i = 0; i < routes.length; i++) {
      if (routes[i] !== Number(startId)) continue;
      if ((routes[i - 1] && routes[i - 1] === endId) || (routes[i + 1] && routes[i + 1] === endId)) {
        pathTimes++;
      }
    }
    if (pathTimes > 1) return 'repeat';
    if (pathTimes === 1) return true;
    return false;
  }

  checkSelectedPath(startPointId, endPointId) {
    if (!this.selectedPaths[startPointId]) return;
    return this.selectedPaths[startPointId].includes(endPointId);
  }

  checkNewPath(startPointId, endPointId) {
    if (!this.newSegmentsCache[startPointId]) return;
    return this.newSegmentsCache[startPointId].includes(endPointId);
  }

  drawSegmentsBatch(segments) {
    segments &&
      Object.keys(segments).forEach(startPointId => {
        segments[startPointId].forEach(endPointId => {
          let color = this.defaultStrokeStyle;
          const p1 = this.scaledPointsHash[startPointId];
          const p2 = this.scaledPointsHash[endPointId];
          if (!p1 || !p2) return;
          if (this.funcSwitch.navigate && this.checkTaskPath(startPointId, endPointId)) color = this.defaultTaskColor; // 当前任务的路径
          if (this.funcSwitch.navigate && this.checkTaskPath(startPointId, endPointId) === 'repeat')
            color = this.defaultRepeatColor; // 当前任务的路径
          if (this.funcSwitch.navigate && this.checkTaskPath(startPointId, endPointId, true))
            color = this.defaultDoneColor; // 当前任务已完成的路径
          if (this.funcSwitch.navigate && this.checkTaskPath(startPointId, endPointId, true) === 'repeat')
            color = this.defaultRepeatDoneColor; // 当前任务已完成的路径
          if (this.checkNewPath(startPointId, endPointId)) color = this.defaultNewPathColor; // 新添加的路径
          if (this.checkSelectedPath(startPointId, endPointId)) color = this.defaultSelectedColor; // 选中的路径
          if (
            !this.funcSwitch.rectSelect &&
            this.hoverPath.startPointId === startPointId &&
            this.hoverPath.endPointId === endPointId &&
            this.hoverPoint === -1
          ) {
            color = this.defaultHoverColor;
            this.drawText({ x: p1.x, y: p1.y }, p1.id);
            this.drawText({ x: p2.x, y: p2.y }, p2.id);
          } // hover 的路径
          this.drawSegment({ x: p1.x, y: p1.y }, { x: p2.x, y: p2.y }, color);
        });
      });
  }

  /**
   * @Description: 绘制页面上的虚线
   * @Param [{x: Number, y: Number}] p1: 虚线的端点
   * @Param [{x: Number, y: Number}] p2: 虚线的端点
   * @Param [Number] lineWidth: 绘制的虚线的宽度
   * @Param [String] color: 绘制虚线的颜色
   */
  drawDottedSegment(lastPointId, p2, lineWidth = this.defaultLineWidth, color = this.defaultNewPathColor) {
    const p1 = this.scaledPointsHash[lastPointId];
    const ctx = this.mapCanvas.getContext('2d');
    ctx.strokeStyle = color;
    ctx.lineWidth = lineWidth;
    ctx.beginPath();
    ctx.setLineDash([5, 5]);
    ctx.moveTo(Math.floor(p1.x) + 0.5, Math.floor(p1.y) + 0.5);
    ctx.lineTo(Math.floor(p2.x) + 0.5, Math.floor(p2.y) + 0.5);
    ctx.stroke();
    ctx.closePath();
  }

  /**
   * @Description: 绘制页面上的矩形
   * @Param [{x: Number, y: Number}] point: 矩形左上角的点
   * @Param [Number] width: 矩形的宽
   * @Param [Number] height: 矩形的高
   * @Param [Number] lineHeight: 矩形的线宽
   * @Param [Number] regionType: 检修区域类型
   * @Param [String] color: 矩形的颜色
   */
  drawRect({ x, y }, width, height, regionType, rectId, lineWidth = this.defaultLineWidth) {
    const ctx = this.mapCanvas.getContext('2d');
    ctx.strokeStyle = this.defaultStrokeStyle;
    ctx.lineWidth = lineWidth;
    if (this.rectSelectMode) ctx.strokeStyle = this.defaultSelectedColor;
    if (this.car.isCarArriving) ctx.strokeStyle = '#e03406';
    if (this.funcSwitch.addRegion && regionType === 2) ctx.strokeStyle = this.defaultNewPathColor;
    if (this.funcSwitch.addRegion && regionType === 'history') ctx.strokeStyle = this.defaultHistoryRegionColor;
    if (this.funcSwitch.addRegion && !this.isDrawingRect && this.hoverRect.id === rectId)
      ctx.strokeStyle = this.defaultHoverColor;
    // canvas 绘制奇数线宽的时候会有 0.5 像素的模糊边界，如果是奇数则从 0.5 位置开始绘制
    ctx.strokeRect(
      Math.floor(x) + (this.defaultLineWidth % 2 ? 0.5 : 0),
      Math.floor(y) + (this.defaultLineWidth % 2 ? 0.5 : 0),
      Math.floor(width),
      Math.floor(height),
    );
  }

  drawRectsBatch() {
    this.rects.forEach(rect => {
      this.drawRect(rect.point, rect.width, rect.height, rect.type, rect.id);

      if (this.funcSwitch.addRegion && !this.isDrawingRect && this.hoverRect.id === rect.id) {
        this.drawText(rect.point, `区域id: ${rect.id}`);
      }
    });
  }

  /**
   * @Description: 绘制小车
   * @Param [{x: Number, y: Number}]: 小车的坐标
   * @Param [Number] width: 小车的宽
   * @Param [Number] height: 小车的高
   * @Param [String] imgUrl: 小车的图片地址
   */
  drawCar(x, y) {
    const ctx = this.mapCanvas.getContext('2d');
    const startPoint = {
      x: (-1 * this.carInfo.width) / 2,
      y: (-1 * this.carInfo.height) / 2,
    };
    ctx.save();
    ctx.translate(x, y);
    ctx.rotate((this.car.deg / 180) * Math.PI);
    ctx.drawImage(
      this.carImage,
      0,
      0,
      this.carImage.naturalWidth,
      this.carImage.naturalHeight,
      startPoint.x,
      startPoint.y,
      this.carInfo.width,
      this.carInfo.height,
    );
    if (this.car.isCarArriving) this.drawRect(startPoint, this.carInfo.width, this.carInfo.height, null, null, 1);
    ctx.restore();
  }
  // 如果当前有任务则连接当前已完成任务的最后一个点和小车
  drawCarBatch() {
    if (this.car.x === -1 || this.car.y === -1) return; // 如果小车坐标没有传入则不渲染
    this.drawCar(this.car.x, this.car.y);

    if (this.donePath.length > 0) {
      const lastDoneTaskPoint = this.scaledPointsHash[this.donePath[this.donePath.length - 1]];
      if (lastDoneTaskPoint) this.drawSegment(this.car, lastDoneTaskPoint, this.defaultDoneColor); // 防止底层传递不存在的点位
    }
  }

  /**
   * @Description: 将地图绘制到 canvas 上
   */
  drawCanvasBg() {
    const ctx = this.mapCanvas.getContext('2d');
    ctx.drawImage(
      this.img,
      0,
      0,
      this.img.naturalWidth,
      this.img.naturalHeight,
      0,
      0,
      this.mapCanvas.width,
      this.mapCanvas.height,
    );
  }

  /**
   * @Description: canvas 上添加每个标注的标注类型
   * @Param:
   */
  drawText(p, text) {
    const ctx = this.mapCanvas.getContext('2d');
    ctx.save();
    ctx.font = 'bold 18px serif';
    ctx.fillStyle = 'black';
    ctx.fillText(text, p.x + 10, p.y);
    ctx.restore();
  }

  /**
   * @Description: 当 canvas 发生变化的时候需要对 canvas 进行重新渲染
   */
  handleReRender() {
    // 清空画布
    const ctx = this.mapCanvas.getContext('2d');
    ctx.clearRect(0, 0, this.mapCanvas.width, this.mapCanvas.height);
    this.initCanvasConf();

    // 重新绘制
    this.drawCanvasBg();
    this.drawPointsBatch(this.points);
    this.drawSegmentsBatch(this.segments);
    this.newSegments.length &&
      this.dashedSegment.x !== -1 &&
      this.drawDottedSegment(this.newSegments[this.newSegments.length - 1], this.dashedSegment);
    this.drawRectsBatch();
    this.funcSwitch.navigate && this.drawCarBatch();
    this.drawPointCloudBatch(this.pointCloud);
  }

  /**
   * @Description: 获取 wrapper 的宽高
   */
  getWrapperSize() {
    return {
      width: getComputedStyle(this.mapWrapper, 'width'),
      height: getComputedStyle(this.mapWrapper, 'height'),
    };
  }

  /**
   * @Description: 监听页面的 resize，重新计算 canvas 大小
   */
  wrapperResizeListener() {
    this.handleResize = throttle(this.calcCanvasSize, 300);
    window.addEventListener('resize', this.handleResize);
    // window.onResize = throttle(this.calcCanvasSize, 16)
  }

  /**
   * @Description: 处理右键菜单（除检修区域和地图选点，其他页面均有该功能，由 showContextmenu prop 控制开关)
   */
  handleContextMenu(e) {
    e.preventDefault(); // 取消右键菜单的默认行为
    this.store.commit('emptyNewPath'); // 结束当前线路规划，和双击效果相同

    if (this.funcSwitch.navigate) {
      this.hoverPoint !== -1 ? (this.navigateId = this.hoverPoint) : (this.navigateId = -1);
    }

    const canvasClientX = this.mapWrapper.getBoundingClientRect().left;
    const canvasClientY = this.mapWrapper.getBoundingClientRect().top;

    const wrapperWidth = getComputedStyle(this.mapWrapper, 'width');
    const wrapperHeight = getComputedStyle(this.mapWrapper, 'height');
    const calcLeft = e.clientX - canvasClientX;
    const calcTop = e.clientY - canvasClientY;
    const contextmenuHeight = this.contextmenuList.length * 32;
    const contextmenuWidth = 100;

    this.contextMenu.style.left = `${
      wrapperWidth - calcLeft > contextmenuWidth ? calcLeft : calcLeft - contextmenuWidth
    }px`;
    this.contextMenu.style.top = `${
      wrapperHeight - calcTop > contextmenuHeight ? calcTop : calcTop - contextmenuHeight
    }px`;
    this.showContextMenu = true;
  }
  bindContextMenuEvent() {
    this.funcSwitch.contextmenu &&
      this.mapWrapper &&
      this.mapWrapper.addEventListener('contextmenu', this.handleContextMenu);
  }
  unbindContextMenuEvent() {
    this.funcSwitch.contextmenu &&
      this.mapWrapper &&
      this.mapWrapper.removeEventListener('contextmenu', this.handleContextMenu);
  }

  // 管理删除状态
  handleStartDel() {
    this.delMode = true;
    this.showContextMenu = false;
  }
  handleDelConfirm() {
    this.delMode = false;
    this.showContextMenu = false;

    Object.keys(this.selectedPaths).forEach(startPointId => {
      this.selectedPaths[startPointId].length &&
        this.selectedPaths[startPointId].forEach(endPointId => {
          this.store.commit('delSegments', startPointId, endPointId);
        });
    });

    this.selectedPoints.forEach(pointId => {
      this.store.commit('delPoint', pointId);
    });
    this.store.commit('emptySelected'); // 完成删除后在清空 selected 的点和线
  }
  handleDelCancel() {
    this.delMode = false;
    this.showContextMenu = false;
    this.store.commit('emptySelected');
  }
  handleNavigate() {
    this.$emit('navigate', this.navigateId);
  }
  handleRelocation() {
    this.relocationMode = true;
  }
  submitRelocation() {
    this.relocationMode = false;
    const point = antiPointsFormatBatch(
      [this.car],
      { x: this.mapInfo.originX, y: this.mapInfo.originY },
      this.mapInfo.proportion,
      this.imageNaturalHeight,
      this.scaleRatio,
    )[0];
    point.deg = this.car.deg;
    this.$emit('relocation', point);
  }
  cancelRelocation() {
    this.relocationMode = false;
    this.carPositionTransform();
  }
  // 关闭右键菜单
  closeContextMenu() {
    this.navigateId = -1;
    this.showContextMenu = false;
  }
  bindCloseDisableContextMenu() {
    this.funcSwitch.contextmenu && document.addEventListener('click', this.closeContextMenu);
  }
  unbindCloseDisableContextMenu() {
    this.funcSwitch.contextmenu && document.removeEventListener('click', this.closeContextMenu);
  }

  /**
   * @Description: 判断是否在当前路径中
   */
  isInCurrentPaths(startPointId, endPointId) {
    if (Number(startPointId) === Number(endPointId)) return true;
    startPointId > endPointId && ([startPointId, endPointId] = [endPointId, startPointId]);
    if (!this.segments[startPointId]) return false;
    const index = this.segments[startPointId].indexOf(endPointId);
    return index !== -1;
  }

  /**
   * @Description: 点位和连线的多选删除和连线操作，当 delMode 为 false 时表示连线状态，当 delMode 为 true 时则为删除点线状态（巡检地图维护）
   */
  handleRouteController() {
    if (this.delMode || this.funcSwitch.rectSelect) {
      if (this.hoverPoint !== -1) {
        // 最后一个参数是判断在地图选点页面不需要选择线，
        this.store.commit('refreshSelectedPoints', this.hoverPoint, this.funcSwitch.rectSelect);
      }

      if (this.hoverPath.startPointId !== -1 && this.hoverPoint === -1 && !this.funcSwitch.rectSelect) {
        if (
          this.selectedPoints.includes(Number(this.hoverPath.startPointId)) ||
          this.selectedPoints.includes(Number(this.hoverPath.endPointId))
        )
          return; // 如果线段的端点已经被选中，则线段不能取消选择
        this.store.commit('refreshSelectedSegments', this.hoverPath.startPointId, this.hoverPath.endPointId);
      }
    } else {
      const length = this.newSegments.length;
      if (
        this.hoverPoint !== -1 &&
        (length === 0 || !this.isInCurrentPaths(this.newSegments[length - 1], this.hoverPoint))
      ) {
        this.store.commit('addNewPath', this.hoverPoint);
      }
    }
  }
  // 在连线状态需要显示鼠标和上一个点的虚线（上一个点不存在则没有该虚线）
  handleDottedLine(e) {
    this.newSegments.length && this.store.commit('addDashedLine', e.offsetX, e.offsetY);
  }
  // 提供给父组件的清空粉红色新路径
  emptyNewPathCache() {
    this.store.commit('emptyNewPathCache');
  }
  // 双击完成路径绘制
  handleRouteCompleted() {
    this.store.commit('emptyNewPath');
  }
  bindContextmenuClick() {
    (this.funcSwitch.route || this.funcSwitch.rectSelect) &&
      this.mapCanvas &&
      this.mapCanvas.addEventListener('click', this.handleRouteController);
    this.funcSwitch.route &&
      this.mapCanvas &&
      this.mapCanvas.addEventListener('mousemove', throttle(this.handleDottedLine, 16));
    this.funcSwitch.route && this.mapCanvas && this.mapCanvas.addEventListener('dblclick', this.handleRouteCompleted);
  }
  unbindContextmenuClick() {
    (this.funcSwitch.route || this.funcSwitch.rectSelect) &&
      this.mapCanvas &&
      this.mapCanvas.removeEventListener('click', this.handleRouteController);
    this.funcSwitch.route &&
      this.mapCanvas &&
      this.mapCanvas.removeEventListener('mousemove', throttle(this.handleDottedLine, 16));
    this.funcSwitch.route &&
      this.mapCanvas &&
      this.mapCanvas.removeEventListener('dblclick', this.handleRouteCompleted);
  }

  /**
   * @description: 处理地图维护页面的导航点添加的功能
   */
  handleAddPointStart() {
    this.addPointMode = true;
  }

  handleAddPointEnd() {
    this.addPointMode = false;
  }

  /**
   * @Description: 处理点线 hover 高亮，除检修区域，其他页面都需要开启
   */
  handleHoverHighlight(e) {
    // 判断小车的 hover
    if (
      this.funcSwitch.navigate &&
      e.offsetX > this.car.x - this.carInfo.width / 2 &&
      e.offsetX < this.car.x + this.carInfo.width / 2 &&
      e.offsetY > this.car.y - this.carInfo.height / 2 &&
      e.offsetY < this.car.y + this.carInfo.height / 2
    ) {
      this.isHoveringCar = true;
    } else {
      this.isHoveringCar = false;
    }

    // 判断点线的 hover
    let hoverId = -1;
    const hoverSegment = { startPointId: -1, endPointId: -1 };
    this.points.forEach(point => {
      if (isInCircle({ x: e.offsetX, y: e.offsetY }, point, this.defaultPointRadius)) {
        hoverId = point.id;
      }
    });
    this.store.commit('refreshHoverPoint', hoverId);

    this.segments &&
      Object.keys(this.segments).forEach(startPointId => {
        this.segments[startPointId].forEach(endPointId => {
          const startPoint = this.scaledPointsHash[startPointId];
          const endPoint = this.scaledPointsHash[endPointId];
          if (
            isInSegment(
              { x: e.offsetX, y: e.offsetY },
              { p1: { x: startPoint.x, y: startPoint.y }, p2: { x: endPoint.x, y: endPoint.y } },
              this.defaultLineWidth,
            )
          ) {
            hoverSegment.startPointId = startPointId;
            hoverSegment.endPointId = endPointId;
          }
        });
      });
    this.store.commit('refreshHoverPath', hoverSegment.startPointId, hoverSegment.endPointId);

    // 判断矩形的 hover
    let hoverRectId = -1;
    this.funcSwitch.addRegion &&
      this.rects.some(rect => {
        if (
          isInRectSide(
            { x: e.offsetX, y: e.offsetY },
            { x: rect.point.x, y: rect.point.y, width: rect.width, height: rect.height },
            this.defaultLineWidth,
          ).res
        ) {
          hoverRectId = rect.id;
          return true;
        }
        return false;
      });
    this.store.commit('refreshHoverRect', hoverRectId);
  }
  bindMousemoveHover() {
    this.mapCanvas && this.mapCanvas.addEventListener('mousemove', throttle(this.handleHoverHighlight, 16));
  }
  unbindMousemoveHover() {
    this.mapCanvas && this.mapCanvas.removeEventListener('mousemove', throttle(this.handleHoverHighlight, 16));
  }

  /**
   * @Description: 重新计算点位坐标，主要是缩放的时候整体绘制内容都要缩放比例重绘
   */
  reCalcPointsCoordinates() {
    const scaledPoints = this.unScaledPoints.map(point => {
      return {
        id: point.id,
        x: point.x * this.scaleRatio,
        y: point.y * this.scaleRatio,
        z: point.z,
        mapId: point.mapId,
      };
    });
    this.store.commit('setData', {
      name: 'points',
      data: scaledPoints,
    });
  }

  // 格式化检修区域的数据到绘制需要个格式
  reCalcRegionsCoordinates() {
    this.store.commit('setData', {
      name: 'rects',
      data: rectFormatBatch(
        this.allRegions,
        { x: this.mapInfo.originX, y: this.mapInfo.originY },
        this.scaleRatio,
        this.mapInfo.proportion,
        this.imageNaturalHeight,
      ),
    });
  }

  /**
   * @Description: 以鼠标为中心进行缩放，当发生缩放时，根据鼠标位置计算 top 和 left
   * @Param [Number] ratioChange: scaleRatio 的表画质
   */
  calcCanvasRelocation(ratioChange) {
    if (Number(Math.abs(ratioChange).toFixed(2)) !== this.scaleRatioChangeStep) return; // 初始化时不进行 canvas 定位
    const widthChange = this.imageNaturalWidth * ratioChange * -1;
    const heightChange = this.imageNaturalHeight * ratioChange * -1;

    const { x: centerX, y: centerY } = this.scaleCenter;
    const widthChangeRatio = centerX / this.mapCanvas.width;
    const heightChangeRatio = centerY / this.mapCanvas.height;

    const canvasLeft = getComputedStyle(this.mapCanvas, 'left');
    const canvasTop = getComputedStyle(this.mapCanvas, 'top');

    // console.log(centerX, centerY);

    switch (this.canvasRotateDeg % 360) {
      case 0:
        this.mapCanvas.style.left = `${canvasLeft + widthChange * widthChangeRatio}px`;
        this.mapCanvas.style.top = `${canvasTop + heightChange * heightChangeRatio}px`;
        break;
      case 90:
        // this.mapCanvas.style.left = `${canvasLeft + heightChange * (1 - heightChangeRatio)}px`
        // this.mapCanvas.style.top = `${canvasTop + widthChange * widthChangeRatio}px`
        // // eslint-disable-next-line no-case-declarations
        // const scaleCenter = {x: centerX, y: centerY};
        // console.log(this.mapCanvas.width, this.mapCanvas.height);
        // // eslint-disable-next-line no-case-declarations
        // const rotateCenter1 = {x: this.mapCanvas.width / 2, y: this.mapCanvas.height / 2}
        // // eslint-disable-next-line no-case-declarations
        // const rotateCenter2 = {x: rotateCenter1.x - widthChange * (0.5 - widthChangeRatio), y: rotateCenter1.y - heightChange * (0.5 - heightChangeRatio)}
        // // eslint-disable-next-line no-case-declarations
        // const {deltaX, deltaY} = getRotateDelta(scaleCenter, rotateCenter1, rotateCenter2);
        // console.log(deltaX, deltaY);
        this.mapCanvas.style.left = `${canvasLeft + widthChange * widthChangeRatio}px`;
        this.mapCanvas.style.top = `${canvasTop + heightChange * heightChangeRatio}px`;
        break;
      case 180:
        this.mapCanvas.style.left = `${canvasLeft + widthChange * (1 - widthChangeRatio)}px`;
        this.mapCanvas.style.top = `${canvasTop + heightChange * (1 - heightChangeRatio)}px`;
        break;
      case 270:
        this.mapCanvas.style.left = `${canvasLeft + widthChange * widthChangeRatio}px`;
        this.mapCanvas.style.top = `${canvasTop + heightChange * heightChangeRatio}px`;
        break;
    }
  }

  /**
   * @Description: 监听滚轮事件缩放地图
   */
  @Watch('scaleRatio')
  onScaleRatioChange(newVal, oldVal) {
    this.reCalcPointsCoordinates();
    this.funcSwitch.addRegion && this.reCalcRegionsCoordinates();
    this.funcSwitch.navigate && this.carPositionTransform();

    // 重新计算 canvas 宽高
    this.calcCanvasRelocation(newVal - oldVal);
    this.mapCanvas.width = this.imageNaturalWidth * newVal;
    this.mapCanvas.height = this.imageNaturalHeight * newVal;

    // 重新计算点云的位置
    this.funcSwitch.navigate && this.pointCloudTransform();

    // this.handleReRender();
  }

  /**
   * @Description: wheel 事件绑定和回调函数，根据滚轮事件调整缩放比例, 重定位模式下的小车转动也在这里处理
   */
  handleScaleMap(e) {
    // 小车重定位的旋转
    if (this.isHoveringCar && this.relocationMode) {
      this.store.commit(
        'refreshCarPosition',
        this.car.x,
        this.car.y,
        (this.car.deg + this.carRotateStep * (e.deltaY >> 31 ? -1 : 1)) % 360,
      );
    }

    // 在重定位和画矩形框的时候不能进行缩放
    if (!this.isDrawingRect && !this.relocationMode) {
      [this.scaleCenter.x, this.scaleCenter.y] = [e.offsetX, e.offsetY];
      if (e.deltaY < 0 && this.scaleRatio > this.minScaleRatio) {
        this.scaleRatio -= this.scaleRatioChangeStep;
      }
      if (e.deltaY > 0 && this.scaleRatio < this.maxScaleRatio) {
        this.scaleRatio += this.scaleRatioChangeStep;
      }
    }
  }
  bindMapScaleListener() {
    this.mapCanvas && this.mapCanvas.addEventListener('wheel', this.handleScaleMap);
  }
  unbindMapScaleListener() {
    this.mapCanvas && this.mapCanvas.removeEventListener('wheel', this.handleScaleMap);
  }

  /**
   * @Description: 处理小车重定位的拖动
   */
  handleRelocationDrag(e) {
    if (!this.isHoveringCar || !this.relocationMode) return;
    const startX = e.offsetX;
    const startY = e.offsetY;

    const originCarX = this.car.x;
    const originCarY = this.car.y;

    const mousemove = e => {
      const offsetX = e.offsetX - startX;
      const offsetY = e.offsetY - startY;
      this.store.commit('refreshCarPosition', originCarX + offsetX, originCarY + offsetY);
    };
    const mouseup = () => {
      this.mapCanvas.removeEventListener('mousemove', mousemove);
      document.removeEventListener('mouseup', mouseup);
    };

    this.mapCanvas.addEventListener('mousemove', mousemove);
    document.addEventListener('mouseup', mouseup);
  }
  bindRelocationDrag() {
    this.mapCanvas && this.mapCanvas.addEventListener('mousedown', this.handleRelocationDrag);
  }
  unbindRelocationDrag() {
    this.mapCanvas && this.mapCanvas.removeEventListener('mousedown', this.handleRelocationDrag);
  }

  /**
   * @Description: 点位框选功能 和 添加检修区域
   */
  handleRectSelectStart() {
    this.rectSelectMode = true;
  }
  handleRectSelectEnd() {
    this.rectSelectMode = false;
  }
  handleAddRegionStart() {
    this.addRegionMode = true;
  }
  handleAddRegionEnd() {
    this.addRegionMode = false;
  }
  // 判断哪些点在框内，加入到 selectedPoints 中
  getSelectedPoints(x, y, width, height, currentSelectedPoints) {
    const minX = x;
    const maxX = x + width;
    const minY = y;
    const maxY = y + height;
    this.points.forEach(point => {
      if (
        point.x > minX &&
        point.x < maxX &&
        point.y > minY &&
        point.y < maxY &&
        !this.selectedPoints.includes(point.id)
      ) {
        // 在矩形范围内且没选中
        this.store.commit('refreshSelectedPoints', point.id, this.funcSwitch.rectSelect, this.isDrawingRect);
      }
      if (
        (point.x < minX || point.x > maxX || point.y < minY || point.y > maxY) &&
        this.selectedPoints.includes(point.id) &&
        !currentSelectedPoints.includes(point.id)
      ) {
        // 在矩形范围外且选中（需要排除在该次框选前就选中的点）
        this.store.commit('refreshSelectedPoints', point.id, this.funcSwitch.rectSelect);
      }
    });
  }
  // 给父组件提供的删除所有选重点的方法
  emptySelectedPoints() {
    this.store.commit('emptySelectedPoints');
  }
  // 转换矩形点的坐标和宽高为后端需要的形式
  transformRect(point, width, height, type) {
    const points = antiPointsFormatBatch(
      [point],
      { x: this.mapInfo.originX, y: this.mapInfo.originY },
      this.mapInfo.proportion,
      this.imageNaturalHeight,
      this.scaleRatio,
    );
    const originWidth = (width / this.scaleRatio) * this.mapInfo.proportion;
    const originHeight = (height / this.scaleRatio) * this.mapInfo.proportion;
    return { vehicleCoordinate: points[0].vehicleCoordinate, width: originWidth, height: originHeight, type };
  }
  handleRectMousemove(e) {
    if (!(this.rectSelectMode || this.addRegionMode)) return;
    this.isDrawingRect = true;
    if (e.button !== 0) return;
    const startX = e.offsetX;
    const startY = e.offsetY;

    // 添加的矩形框信息
    let point;
    let width;
    let height;
    const type = !this.addRegionMode ? undefined : this.regionType;

    const rectNum = this.rects.length;
    const currentSelectedPoint = [...this.selectedPoints]; // 缓存一下框选前已选中的点

    const mousemove = throttle(e => {
      this.rects.length !== rectNum && this.store.commit('delRect'); // 防止误删上一个矩形
      const endX = e.offsetX;
      const endY = e.offsetY;
      const x = Math.min(startX, endX);
      const y = Math.min(startY, endY);
      point = { x, y };
      width = Math.abs(startX - endX);
      height = Math.abs(startY - endY);
      !this.addRegionMode && this.getSelectedPoints(x, y, width, height, currentSelectedPoint); // 只有框选点位的时候才需要选点
      this.store.commit('addRect', { point, width, height, type });
    }, 16);

    const mouseup = () => {
      this.mapCanvas.removeEventListener('mousemove', mousemove);
      document.removeEventListener('mouseup', mouseup);
      // 由于 mousemove 加了节流，所以可能导致 mouseup 后仍然有一次 mousemove，此处用 setTimeout 处理，注意：解绑不能放到定时器中
      setTimeout(() => {
        this.isDrawingRect = false;
        if (width && height) {
          !this.funcSwitch.addRegion && this.store.commit('delRect'); // 地图选点页面需要在 mouseup 时去除矩形框
          this.funcSwitch.addRegion && this.$emit('add-rect', this.transformRect(point, width, height, type));
        }
      }, 50);
    };

    this.mapCanvas.addEventListener('mousemove', mousemove);
    document.addEventListener('mouseup', mouseup);
  }
  bindRectSelectMousemove() {
    this.mapCanvas && this.mapCanvas.addEventListener('mousedown', this.handleRectMousemove);
  }
  unbindRectSelectMousemove() {
    this.mapCanvas && this.mapCanvas.removeEventListener('mousedown', this.handleRectMousemove);
  }

  /**
   * @Description: 处理 canvas 的拖动
   */
  handleDragMouseDown(e) {
    e.preventDefault(); // 防止拖动时选中页面的文本等内容导致拖动收到影响
    if (e.button !== 0 || this.rectSelectMode || this.addRegionMode || this.isHoveringCar || this.relocationMode)
      return;
    const dragBorderDistance = 300; // 拖动 canvas 的边界距离

    let lastClientX = e.clientX;
    let lastClientY = e.clientY;
    // this.cursorState = 'grab'

    // TODO: 此处的拖动边界逻辑判断有些复杂，暂时没有想到更优化的逻辑
    const mousemove = throttle(e => {
      this.cursorState = 'grab';
      const canvasTop = getComputedStyle(this.mapCanvas, 'top');
      const canvasLeft = getComputedStyle(this.mapCanvas, 'left');
      const wrapperWidth = Math.ceil(getComputedStyle(this.mapWrapper, 'width'));
      const wrapperHeight = Math.ceil(getComputedStyle(this.mapWrapper, 'height'));
      const canvasCenterX = this.mapCanvas.offsetLeft + this.mapCanvas.width / 2;
      const canvasCenterY = this.mapCanvas.offsetTop + this.mapCanvas.height / 2;

      // 判断拖动方向
      const { horizontalDragDirection, verticalDragDirection } = getDragDirection(
        e.clientX,
        lastClientX,
        e.clientY,
        lastClientY,
      );

      // 设置拖动范围，注意 transform rotate 导致的垂直拖动需要特殊处理（我们所取到的所有 css 属性都是旋转前的，rotate只是旋转了坐标系），主要思路是以旋转中心为基准点计算出边界（旋转中心默认为 center)
      if (this.canvasRotateDeg % 180) {
        if (
          !(
            (wrapperWidth - canvasCenterX + this.mapCanvas.height / 2 < dragBorderDistance &&
              horizontalDragDirection === 'right') ||
            (canvasCenterX + this.mapCanvas.height / 2 < dragBorderDistance && horizontalDragDirection === 'left')
          )
        ) {
          this.mapCanvas.style.left = `${Math.floor(canvasLeft + e.clientX - lastClientX)}px`;
        }
        if (
          !(
            (wrapperHeight - canvasCenterY + this.mapCanvas.width / 2 < dragBorderDistance &&
              verticalDragDirection === 'bottom') ||
            (canvasCenterY + this.mapCanvas.width / 2 < dragBorderDistance && verticalDragDirection === 'top')
          )
        ) {
          this.mapCanvas.style.top = `${Math.floor(canvasTop + e.clientY - lastClientY)}px`;
        }
      } else {
        if (
          !(
            (this.mapCanvas.offsetLeft < dragBorderDistance - this.mapCanvas.width &&
              horizontalDragDirection === 'left') ||
            (this.mapCanvas.offsetLeft > wrapperWidth - dragBorderDistance && horizontalDragDirection === 'right')
          )
        ) {
          this.mapCanvas.style.left = `${Math.floor(canvasLeft + e.clientX - lastClientX)}px`;
        }
        if (
          !(
            (this.mapCanvas.offsetTop < dragBorderDistance - this.mapCanvas.height &&
              verticalDragDirection === 'top') ||
            (this.mapCanvas.offsetTop > wrapperHeight - dragBorderDistance && verticalDragDirection === 'bottom')
          )
        ) {
          this.mapCanvas.style.top = `${Math.floor(canvasTop + e.clientY - lastClientY)}px`;
        }
      }

      lastClientX = e.clientX;
      lastClientY = e.clientY;
    }, 16);

    const mouseup = () => {
      document.removeEventListener('mousemove', mousemove);
      document.removeEventListener('mouseup', mouseup);
      // mousemove 用了节流，mouseup 需要用一个延迟确保在 mousemove 之后触发，注意：解绑不能放到定时器中
      setTimeout(() => {
        this.cursorState = '';
      }, 50);
    };

    document.addEventListener('mousemove', mousemove);
    document.addEventListener('mouseup', mouseup);
  }
  bindCanvasDragListener() {
    this.mapWrapper && this.mapWrapper.addEventListener('mousedown', this.handleDragMouseDown);
  }
  unbindCanvasDragListener() {
    this.mapWrapper && this.mapWrapper.removeEventListener('mousedown', this.handleDragMouseDown);
  }

  /**
   * @Description: 由于请求回来的导航点需要经过比较复杂的计算才能生成 canvas 渲染需要的坐标，这里单独写一个函数，主要在 allPoints 发生变法的时候使用
   */
  pointsDataInitFormat() {
    // 格式化请求回来的导航点坐标到绘制 canvas 需要的格式, 这里需要将原始数据保存，因为我们的缩放比例都是以原始大小进行计算的
    this.unScaledPoints = pointsFormatBatch(
      this.allPoints,
      { x: this.mapInfo.originX, y: this.mapInfo.originY },
      this.mapInfo.proportion,
      this.imageNaturalHeight,
    );
    this.reCalcPointsCoordinates();
  }

  /**
   * @Description: 初始化绘制点位和路径
   */
  initData() {
    this.pointsDataInitFormat();
    this.store.commit('setData', { name: 'segments', data: routeArrToObj(this.allRoutes) });
    // this.store.commit('setData', {name: 'rects', data: rectFormatBatch(this.allRegions, {x: this.mapInfo.originX, y: this.mapInfo.originY}, this.mapInfo.proportion, this.imageNaturalHeight)})

    // 由于 selectedPaths 的属性都是动态添加，需要用 $set 进行监听
    this.store.commit('observeSelectedSegments', this);
  }

  /**
   * @Description: 根据 wrapper 的宽高和图片的宽高来计算 canvas 的宽高，根据 wrapper 和图片的纵横比决定是用 wrapper 的宽还是高
   */
  calcCanvasSize() {
    const { width: wrapperWidth, height: wrapperHeight } = this.getWrapperSize();
    const wrapperAspectRatio = wrapperWidth / wrapperHeight;
    const imageAspectRatio = this.imageNaturalWidth / this.imageNaturalHeight;

    // 如果图片小于 wrapper 大小，则图片为默认大小
    if (this.imageNaturalHeight < wrapperHeight && this.imageNaturalWidth < wrapperWidth) {
      this.mapCanvas.height = this.imageNaturalHeight;
      this.mapCanvas.width = this.imageNaturalWidth;
      this.scaleRatio = 1;
      this.drawCanvasBg();
      return;
    }

    // 如果图片的宽或者高大于 wrapper，则要根据纵横比进行缩放并计算 scaleRatio
    if (wrapperAspectRatio > imageAspectRatio) {
      this.mapCanvas.height = Math.floor(wrapperHeight);
      this.mapCanvas.width = Math.floor(this.imageNaturalWidth * (wrapperHeight / this.imageNaturalHeight));
      this.scaleRatio = wrapperHeight / this.imageNaturalHeight;
    } else {
      this.mapCanvas.width = Math.floor(wrapperWidth);
      this.mapCanvas.height = Math.floor(this.imageNaturalHeight * (wrapperWidth / this.imageNaturalWidth));
      this.scaleRatio = wrapperWidth / this.imageNaturalWidth;
    }
    this.drawCanvasBg();
  }

  /**
   * @Description: 计算 canvas 的初始位置
   */
  initCanvas() {
    this.calcCanvasSize();

    const { width: wrapperWidth, height: wrapperHeight } = this.getWrapperSize();
    const canvasWidthOverflow = Math.floor(wrapperWidth - this.mapCanvas.width);
    const canvasHeightOverflow = Math.floor(wrapperHeight - this.mapCanvas.height);

    this.mapCanvas.style.left = `${canvasWidthOverflow / 2}px`;
    this.mapCanvas.style.top = `${canvasHeightOverflow / 2}px`;
  }

  /**
   * @Description: 转换小车真实坐标到绘制坐标
   */
  carPositionTransform() {
    if (!this.isCarLoaded || !this.carPosition || !this.carPosition.vehicleCoordinate) return; // 确保小车图片未加载之前不渲染，否则可能导致页面初次加载的时候所有的 drawCar 发生在小车图片 onload 之前
    const transformedCarPosition = pointsFormatBatch(
      [this.carPosition],
      { x: this.mapInfo.originX, y: this.mapInfo.originY },
      this.mapInfo.proportion,
      this.imageNaturalHeight,
    )[0];
    this.store.commit(
      'refreshCarPosition',
      transformedCarPosition.x * this.scaleRatio,
      transformedCarPosition.y * this.scaleRatio,
      (-this.carPosition.deg / Math.PI) * 180,
    );
  }

  /**
   * @Description: 转换点云坐标到绘制坐标, 计算方式为点云的角度加上小车的角度计算
   */
  pointCloudTransform() {
    if (this.car.x === -1) return;
    const transformedPointCloud = this.originPointCloud.map(point => {
      const radian = -point[0] + (this.car.deg / 180) * Math.PI;
      const distance = point[1];
      const deltaX = ((distance * Math.cos(radian)) / this.mapInfo.proportion) * this.scaleRatio;
      const deltaY = ((distance * Math.sin(radian)) / this.mapInfo.proportion) * this.scaleRatio;
      return {
        x: this.car.x + deltaX,
        y: this.car.y + deltaY,
      };
    });
    this.store.commit('setData', { name: 'pointCloud', data: transformedPointCloud });
  }

  /**
   * @Description: 初始化小车位置并渲染
   */
  initCar() {
    if (this.funcSwitch.navigate) {
      this.carImage = new Image();
      this.carImage.src = this.carInfo.imgUrl;

      this.carImage.onload = () => {
        this.isCarLoaded = true;
        this.carPositionTransform();
        this.pointCloudTransform();
      };
    }
  }

  /**
   * @Description: 加载图片，在图片加载完成后绘制图片到 canvas
   * @Param [String] imgUrl: 地图 url
   */
  loadImage() {
    // this.scaleRatio = 1;
    return new Promise((resolve, reject) => {
      if (!this.mapInfo.address) return reject('地图尚未加载');
      this.img = new Image();
      this.img.src = `${process.env.VUE_APP_SOURCE_STATIC}${this.mapInfo.address}`;
      this.img.onload = () => {
        this.imageNaturalWidth = this.img.naturalWidth;
        this.imageNaturalHeight = this.img.naturalHeight;
        this.initCanvas(); // 渲染 canvas
        this.initData(); // 加载数据
        this.funcSwitch.navigate && this.initCar(); // 初始化小车
        resolve('canvas 初始渲染完毕，开始绑定事件');
      };
    });
  }

  bindAllListener() {
    this.bindMapScaleListener();
    this.bindCanvasDragListener();
    this.bindMousemoveHover();
    this.bindContextMenuEvent();
    this.bindCloseDisableContextMenu();
    this.bindContextmenuClick(); // 此处分号不能省略
    (this.funcSwitch.rectSelect || this.funcSwitch.addRegion) && this.bindRectSelectMousemove();
    this.funcSwitch.navigate && this.bindRelocationDrag();
    // this.wrapperResizeListener()
  }

  // 解绑所有事件
  unbindAllListener() {
    this.unbindMapScaleListener();
    this.unbindCanvasDragListener();
    this.unbindMousemoveHover();
    this.unbindContextMenuEvent();
    this.unbindCloseDisableContextMenu();
    this.unbindContextmenuClick(); // 此处分号不能省略
    (this.funcSwitch.rectSelect || this.funcSwitch.addRegion) && this.unbindRectSelectMousemove();
    this.funcSwitch.navigate && this.unbindRelocationDrag();
    // this.wrapperResizeListener()
  }

  // loadImage 并绑定所有事件
  initComponent() {
    this.loadImage().then(
      resolveMsg => {
        console.log(resolveMsg);
        this.bindAllListener();
      },
      rejectMsg => {
        console.log(rejectMsg);
      },
    );
  }

  mounted() {
    this.initComponent();
  }

  beforeDestroy() {
    this.unbindAllListener();
    //window.removeEventListener('resize', this.handleResize);
  }
}
</script>

<style lang="scss" scoped>
.wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;

  .map-canvas {
    position: absolute;
    top: 0;
    left: 0;
    //transform-origin: center;
  }

  .context-menu {
    position: absolute;
    top: 0;
    left: 0;
    display: none;
    border: 2px solid #670909;

    .item {
      width: 100px;
      height: 32px;
      padding: 5px 8px;
      line-height: 22px;
      text-align: center;
      font-weight: 700;
      color: #670909;
      cursor: pointer;
    }

    .item:not(:last-child) {
      border-bottom: 2px solid #670909;
    }

    &.active {
      display: flex;
      flex-direction: column;
    }
  }
}

.wrapper.grab {
  cursor: grab;
}

.tools {
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
}

.spin-canvas,
.show-task-path {
  display: flex;
  justify-content: center;
  align-items: center;
  //margin-left: -16px;
  width: 32px;
  height: 32px;
  border: 1px solid;
  font-size: 20px;
  cursor: pointer;
}

.legends {
  position: absolute;
  top: 10px;
  left: -140px;
  width: 140px;
  display: flex;
  flex-direction: column;
  padding: 5px;
  border: 1px solid rgba(255, 255, 255, 0.5);
  border-radius: 5px;
  background-color: rgba(0,0,0,0.5);
  transition: all 0.3s;

  .title {
    color: #34e8cd;
    font-weight: 700;
  }

  .legend {
    display: flex;
    align-items: center;
    color: white;
    font-size: 12px;

    .color {
      width: 8px;
      height: 8px;
      border-radius: 50%;
      margin-right: 5px;
    }
  }

  .car-realtime-info {
    display: flex;
    font-size: 12px;
    color: white;

    .name {
      margin-right: 5px;
    }
  }

  .slider {
    position: absolute;
    left: 100%;
    top: 50%;
    border: 1px solid rgba(255, 255, 255, 0.5);
    border-left: none;
    border-radius: 0 5px 5px 0;
    color: rgba(255, 255, 255, 0.5);
    transform: translateY(-50%);
  }
}
.legends.show-legends {
  left: 0;
}

.task-paths {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  padding: 10px;
  border: 1px solid;
  background: white;
}

.one-key {
  position: absolute;
  top: 10px;
  right: 10px;
  padding: 5px;
  border-radius: 5px;
  border: 1px solid #fb516e;
  color: #fb516e;
  animation: 1.5s linear infinite fade;
}
@keyframes fade {
  35% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
  65% {
    opacity: 1;
  }
}
</style>
