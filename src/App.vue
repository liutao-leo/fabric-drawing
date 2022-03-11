<template>
  <div class="maintenancePlanAdd">
    <div class="child-panel-title">
      <h2>图形标注</h2>
    </div>
    <div class="panel-body">
      <div class="demo">
        <canvas id="canvas" :width="width" :height="height"></canvas>
        <div class="draw-btn-group">
          <div
            :class="{ active: drawType == '' }"
            title="自由选择"
            @click="drawTypeChange('')"
          >
            <i class="draw-icon icon-mouse"></i>
          </div>
          <div
            :class="{ active: drawType == 'ellipse' }"
            title="画圆"
            @click="drawTypeChange('ellipse')"
          >
            <i class="draw-icon icon-3"></i>
          </div>
          <div
            :class="{ active: drawType == 'rectangle' }"
            title="画矩形"
            @click="drawTypeChange('rectangle')"
          >
            <i class="draw-icon icon-4"></i>
          </div>
          <div
            :class="{ active: drawType == 'polygon' }"
            title="画多边形"
            @click="drawPolygon"
          >
            <i class="draw-icon icon-6"></i>
          </div>
          <div
            :class="{ active: drawType == 'pen' }"
            title="笔画"
            @click="drawTypeChange('pen')"
          >
            <i class="draw-icon icon-7"></i>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  name: 'App',
  data() {
    return {
      width: 1280,
      height: 720,
      rect: [],
      canvas: {},
      showMenu: false,
      x: '',
      y: '',

      mouseFrom: {},
      mouseTo: {},
      drawType: null, //当前绘制图像的种类
      canvasObjectIndex: 0,
      rectangleLabel: 'warning',
      drawWidth: 2, //笔触宽度
      color: '#E34F51', //画笔颜色
      drawingObject: null, //当前绘制对象
      moveCount: 1, //绘制移动计数器
      doDrawing: false, // 绘制状态

      //polygon 相关参数
      polygonMode: false,
      pointArray: [],
      lineArray: [],
      activeShape: false,
      activeLine: '',
      line: {},

      delectKlass: {},
    }
  },
  watch: {
    drawType() {
      this.canvas.selection = !this.drawType
    },
    width() {
      this.canvas.setWidth(this.width)
    },
    height() {
      this.canvas.setHeight(this.height)
    },
  },
  methods: {
    // 开始绘制时，指定绘画种类
    drawTypeChange(e) {
      this.drawType = e
      this.canvas.skipTargetFind = !!e
      if (e == 'pen') {
        // isDrawingMode为true 才可以自由绘画
        this.canvas.isDrawingMode = true
        this.canvas.freeDrawingBrush.color = 'green'
        this.canvas.freeDrawingBrush.width = 2
      } else {
        this.canvas.isDrawingMode = false
      }
    },
    // 鼠标按下时触发
    mousedown(e) {
      // 记录鼠标按下时的坐标
      var xy = e.pointer || this.transformMouse(e.e.offsetX, e.e.offsetY)
      this.mouseFrom.x = xy.x
      this.mouseFrom.y = xy.y
      this.doDrawing = true
      const activeObj = this.canvas.getActiveObjects()
      console.log(xy, activeObj)
      if (activeObj.length !== 0) {
        let points = []
        switch (activeObj[0].name) {
          case 'ellipse':
            const params = {
              a: activeObj[0].rx * activeObj[0].scaleX,
              b: activeObj[0].ry * activeObj[0].scaleY,
              p: 1,
              cx: activeObj[0].translateX,
              cy: activeObj[0].translateY,
              angle: activeObj[0].angle ? activeObj[0].angle : 0,
            }
            points = this.getCirPoint(params)
            break
          case 'rectangle':
            // 按顺序
            points.push([activeObj[0].aCoords.tl.x, activeObj[0].aCoords.tl.y])
            points.push([activeObj[0].aCoords.tr.x, activeObj[0].aCoords.tr.y])
            points.push([activeObj[0].aCoords.br.x, activeObj[0].aCoords.br.y])
            points.push([activeObj[0].aCoords.bl.x, activeObj[0].aCoords.bl.y])
            break
          case 'polygon':
            activeObj[0].points.map(item => {
              points.push([item.x, item.y])
            })
            break
        }
        console.log(points)
      }
      // 绘制多边形
      if (this.drawType == 'polygon') {
        this.canvas.skipTargetFind = false
        try {
          // 此段为判断是否闭合多边形，点击红点时闭合多边形
          if (this.pointArray.length > 1) {
            // e.target.id == this.pointArray[0].id 表示点击了初始红点
            if (e.target && e.target.id == this.pointArray[0].id) {
              this.generatePolygon()
            }
          }
          //未点击红点则继续作画
          if (this.polygonMode) {
            this.addPoint(e)
          }
        } catch (error) {
          console.log(error)
        }
      }
    },
    // 鼠标松开执行
    mouseup(e) {
      var xy = e.pointer || this.transformMouse(e.e.offsetX, e.e.offsetY)
      this.mouseTo.x = xy.x
      this.mouseTo.y = xy.y
      this.drawingObject = null
      this.moveCount = 1
      if (this.drawType != 'polygon') {
        this.doDrawing = false
      }
    },

    //鼠标移动过程中已经完成了绘制
    mousemove(e) {
      if (this.moveCount % 2 && !this.doDrawing) {
        //减少绘制频率
        return
      }
      this.moveCount++
      var xy = e.pointer || this.transformMouse(e.e.offsetX, e.e.offsetY)
      this.mouseTo.x = xy.x
      this.mouseTo.y = xy.y
      // 多边形特殊处理
      if (this.drawType != 'polygon') {
        this.drawing(e)
      }
      if (this.drawType == 'polygon') {
        if (this.activeLine && this.activeLine.class == 'line') {
          var pointer = this.canvas.getPointer(e.e)
          this.activeLine.set({ x2: pointer.x, y2: pointer.y })

          var points = this.activeShape.get('points')
          points[this.pointArray.length] = {
            x: pointer.x,
            y: pointer.y,
            zIndex: 1,
          }
          this.activeShape.set({
            points: points,
          })
          this.canvas.renderAll()
        }
        this.canvas.renderAll()
      }
    },
    deleteObj() {
      this.canvas.getActiveObjects().map(item => {
        this.canvas.remove(item)
      })
    },
    transformMouse(mouseX, mouseY) {
      return { x: mouseX / 1, y: mouseY / 1 }
    },
    // 绘制多边形开始，绘制多边形和其他图形不一样，需要单独处理
    drawPolygon() {
      this.drawType = 'polygon'
      this.polygonMode = true
      //这里画的多边形，由顶点与线组成
      this.pointArray = new Array() // 顶点集合
      this.lineArray = new Array() //线集合
      this.canvas.isDrawingMode = false
    },
    addPoint(e) {
      var random = Math.floor(Math.random() * 10000)
      var id = new Date().getTime() + random
      var circle = new fabric.Circle({
        radius: 5,
        fill: '#ffffff',
        stroke: '#333333',
        strokeWidth: 0.5,
        left: (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
        top: (e.pointer.y || e.e.layerY) / this.canvas.getZoom(),
        selectable: false,
        hasBorders: false,
        // hasControls: false,
        originX: 'center',
        originY: 'center',
        id: id,
        objectCaching: false,
      })
      if (this.pointArray.length == 0) {
        circle.set({
          fill: 'green',
        })
      }
      var points = [
        (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
        (e.pointer.y || e.e.layerY) / this.canvas.getZoom(),
        (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
        (e.pointer.y || e.e.layerY) / this.canvas.getZoom(),
      ]

      this.line = new fabric.Line(points, {
        strokeWidth: 2,
        fill: '#999999',
        stroke: '#999999',
        class: 'line',
        originX: 'center',
        originY: 'center',
        selectable: false,
        hasBorders: false,
        // hasControls: false,
        evented: false,

        objectCaching: false,
      })
      if (this.activeShape) {
        var pos = this.canvas.getPointer(e.e)
        var points = this.activeShape.get('points')
        points.push({
          x: pos.x,
          y: pos.y,
        })
        var polygon = new fabric.Polygon(points, {
          stroke: '#333333',
          strokeWidth: 1,
          fill: '#cccccc',
          opacity: 0.3,

          selectable: false,
          hasBorders: false,
          // hasControls: false,
          evented: false,
          objectCaching: false,
        })
        this.canvas.remove(this.activeShape)
        this.canvas.add(polygon)
        this.activeShape = polygon
        this.canvas.renderAll()
      } else {
        var polyPoint = [
          {
            x: (e.pointer.x || e.e.layerX) / this.canvas.getZoom(),
            y: (e.pointer.y || e.e.layerY) / this.canvas.getZoom(),
          },
        ]
        var polygon = new fabric.Polygon(polyPoint, {
          stroke: '#333333',
          strokeWidth: 1,
          fill: '#cccccc',
          opacity: 0.3,

          selectable: false,
          hasBorders: false,
          // hasControls: false,
          evented: false,
          objectCaching: false,
        })
        this.activeShape = polygon
        this.canvas.add(polygon)
      }
      this.activeLine = this.line

      this.pointArray.push(circle)
      this.lineArray.push(this.line)
      this.canvas.add(this.line)
      this.canvas.add(circle)
    },
    generatePolygon() {
      var points = new Array()
      this.pointArray.map((point, index) => {
        points.push({
          x: point.left,
          y: point.top,
        })
        this.canvas.remove(point)
      })
      this.lineArray.map((line, index) => {
        this.canvas.remove(line)
      })
      this.canvas.remove(this.activeShape).remove(this.activeLine)
      var polygon = new fabric.Polygon(points, {
        stroke: this.color,
        strokeWidth: this.drawWidth,
        fill: 'rgba(255, 255, 255, 0)',
        opacity: 1,
        hasBorders: true,
        // hasControls: false,
        name: 'polygon',
      })
      this.canvas.add(polygon)

      this.activeLine = null
      this.activeShape = null
      this.polygonMode = false
      this.doDrawing = false
      this.drawType = null
    },
    drawing(e) {
      if (this.drawingObject) {
        this.canvas.remove(this.drawingObject)
      }
      var canvasObject = null
      var left = this.mouseFrom.x,
        top = this.mouseFrom.y,
        mouseFrom = this.mouseFrom,
        mouseTo = this.mouseTo
      switch (this.drawType) {
        case 'ellipse': //椭圆
          // 按shift时画正圆，只有在鼠标移动时才执行这个，所以按了shift但是没有拖动鼠标将不会画圆
          if (e.e.shiftKey) {
            mouseTo.x - left > mouseTo.y - top
              ? (mouseTo.y = top + mouseTo.x - left)
              : (mouseTo.x = left + mouseTo.y - top)
          }
          var radius =
            Math.sqrt(
              (mouseTo.x - left) * (mouseTo.x - left) +
                (mouseTo.y - top) * (mouseTo.y - top)
            ) / 2
          canvasObject = new fabric.Ellipse({
            left: (mouseTo.x - left) / 2 + left,
            top: (mouseTo.y - top) / 2 + top,
            stroke: this.color,
            fill: 'rgba(255, 255, 255, 0)',
            originX: 'center',
            originY: 'center',
            rx: Math.abs(left - mouseTo.x) / 2,
            ry: Math.abs(top - mouseTo.y) / 2,
            strokeWidth: this.drawWidth,
            name: 'ellipse',
          })
          break
        case 'rectangle': //长方形
          // 按shift时画正方型
          if (e.e.shiftKey) {
            mouseTo.x - left > mouseTo.y - top
              ? (mouseTo.y = top + mouseTo.x - left)
              : (mouseTo.x = left + mouseTo.y - top)
          }
          var path =
            'M ' +
            mouseFrom.x +
            ' ' +
            mouseFrom.y +
            ' L ' +
            mouseTo.x +
            ' ' +
            mouseFrom.y +
            ' L ' +
            mouseTo.x +
            ' ' +
            mouseTo.y +
            ' L ' +
            mouseFrom.x +
            ' ' +
            mouseTo.y +
            ' L ' +
            mouseFrom.x +
            ' ' +
            mouseFrom.y +
            ' z'
          canvasObject = new fabric.Path(path, {
            left: left,
            top: top,
            stroke: this.color,
            strokeWidth: this.drawWidth,
            fill: 'rgba(255, 255, 255, 0)',
            // hasControls: false,
            name: 'rectangle',
          })
          //也可以使用fabric.Rect
          break

        default:
          break
      }

      if (canvasObject) {
        // canvasObject.index = getCanvasObjectIndex();\
        this.canvas.add(canvasObject) //.setActiveObject(canvasObject)
        this.drawingObject = canvasObject
      }
    },
    //椭圆坐标
    // a x半径, b y半径, p 节点的间隔, cx, cy 圆心, angle 旋转角度
    getCirPoint({ a, b, p = 1, cx = 0, cy = 0, angle = 0 }) {
      const data = []
      for (let index = 0; index < 360; index = index + p) {
        const x = cx + a * Math.cos((Math.PI * 2 * index) / 360)
        const y = cy + b * Math.sin((Math.PI * 2 * index) / 360)
        const x0 =
          cx +
          (x - cx) * Math.cos((angle * Math.PI) / 180) -
          (y - cy) * Math.sin((angle * Math.PI) / 180)
        const y0 =
          cy +
          (x - cx) * Math.sin((angle * Math.PI) / 180) +
          (y - cy) * Math.cos((angle * Math.PI) / 180)
        data.push([x0, y0])
      }
      return data
    },
  },
  mounted() {
    this.canvas = new fabric.Canvas('canvas', {})
    this.canvas.selectionColor = 'rgba(0,0,0,0.05)'
    this.canvas.on('mouse:down', this.mousedown)
    this.canvas.on('mouse:move', this.mousemove)
    this.canvas.on('mouse:up', this.mouseup)

    document.onkeydown = e => {
      // 键盘 delete删除所选元素
      if (e.keyCode === 8) {
        this.deleteObj()
      }
      // ctrl+z 删除最近添加的元素
      if (e.keyCode === 90 && e.ctrlKey) {
        this.canvas.remove(
          this.canvas.getObjects()[this.canvas.getObjects().length - 1]
        )
      }
    }
  },
}
</script>

<style lang="scss" scoped>
.el-container {
  flex-direction: column;
}
.demo {
  display: flex;
  flex-direction: column;
  align-items: center;
}
canvas {
  border: 1px dashed black;
}
.draw-btn-group {
  // width: 1270px;
  margin-top: 10px;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  & > div {
    background: #fafafa;
    cursor: pointer;
    &:hover {
      background: #eee;
    }
    i {
      display: flex;
      background-repeat: no-repeat;
      background-size: 80%;
      background-position: 50% 50%;
      height: 30px;
      width: 30px;
    }
    .icon-3 {
      background-image: url('./assets/icons/draw/3.png');
    }
    .icon-4 {
      background-image: url('./assets/icons/draw/4.png');
      background-size: 75%;
    }
    .icon-5 {
      background-image: url('./assets/icons/draw/5.png');
      background-size: 70%;
    }
    .icon-6 {
      background-image: url('./assets/icons/draw/6.png');
    }
    .icon-7 {
      background-image: url('./assets/icons/draw/7.png');
      background-size: 80%;
    }
    .icon-del {
      background-image: url('./assets/icons/draw/del.png');
      background-size: 90%;
    }
    .icon-back {
      background-image: url('./assets/icons/draw/back.png');
      background-size: 75%;
    }
    .icon-mouse {
      background-image: url('./assets/icons/draw/mouse.png');
      background-size: 60%;
    }
  }
  .active {
    background: #eee;
  }
}
</style>
