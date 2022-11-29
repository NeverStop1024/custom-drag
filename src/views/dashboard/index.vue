<template>
  <div class="dashboard-container">
    <div style="width: 200px;height: 50px;display: flex">
      <column/>
      <line-chart/>
    </div>
    <div class="dashboard-text">
      <!--使用draggable组件-->
      <div class="itxst">
        <ul class="col toolsBox">
          <!-- 拖出 者 -->
          <li v-for="item in arr1" :key="item.id" class="li" draggable="true" @dragstart="dragStartFn(item.id)" @dragend="dragEnd()">
            <div class="item">{{ item.name }}</div>
          </li>
          <li class="save" @click="saveWidgetsFn">
            保存
          </li>
        </ul>
        <!-- S 展示台 -->
        <div class="col big-father">
          <!-- 标尺 -->
          <vue-ruler-tool
            v-model="dashboard.presetLine"
            class="vueRuler"
            :step-length="50"
            :parent="true"
            :position="'relative'"
            :is-scale-revise="true"
            :visible.sync="dashboard.presetLineVisible"
          >
            <!-- 工作台 -->
            <div
              id="workbench"
              class="workbench"
              @drop="widgetOnDraggedFn($event)"
              @dragover="dragOverFn($event)"
              @mousedown.prevent="handleMousedown"
            >
              <avue-draggable
                v-for="(item2,index2) in widgets"
                :key="index2"
                ref="draggableAvue"
                style="position: absolute;"
                :index="index2"
                :width="item2.value.width"
                :height="item2.value.height"
                :left="item2.value.left"
                :top="item2.value.top"
                @focus="handleFocus"
                @blur="handleBlur"
              >
                <component :is="item2.id"></component>
              </avue-draggable>
            </div>
          </vue-ruler-tool>
        </div>
      </div>
    </div>
  </div></template>

<script>
import VueRulerTool from 'vue-ruler-tool'
import Column from "./components/Column";
import LineChart from "./components/LineChart";

export default {
  name: 'Dashboard',
  components: {
    VueRulerTool,
    Column,
    LineChart
  },
  data() {
    return {
      // 定义要被拖拽对象的数组
      arr1: [
        { id: 'Column', name: '柱状图' },
        { id: 'LineChart', name: '折线图' }
      ],
      dragWidgetId: '', // 当前从工具栏拖拽的组件种类Id
      currentIndex: '', // 当前工作台上操作的组件index
      // 工作台大屏画布，保存到表gaea_report_dashboard中
      dashboard: {
        id: null,
        title: '', // 大屏页面标题
        width: 1920, // 大屏设计宽度
        height: 1080, // 大屏设计高度
        backgroundColor: '', // 大屏背景色
        backgroundImage: '', // 大屏背景图片
        refreshSeconds: null, // 大屏刷新时间间隔
        presetLine: [], // 辅助线
        presetLineVisible: true // 辅助线是否显示
      },
      // 大屏画布中的组件
      widgets: [
        // {
        //   // type和value最终存到数据库中去，保存到gaea_report_dashboard_widget中
        //   id: '',
        //   value: {
        //     width: 200,
        //     height: 200,
        //     left: 200,
        //     top: 200
        //   }
        // }
      ] // 工作区中拖放的组件
    }
  },
  created() {
    if (JSON.parse(localStorage.getItem('saveWidgetsFn'))) {
      this.widgets = JSON.parse(localStorage.getItem('saveWidgetsFn'))
    }
  },
  methods: {
    // tool类拖动标准
    dragStartFn(id) {
      console.log('拖拽组件id为', id)
      this.dragWidgetId = id
    },
    dragEnd() {
      console.log('拖拽完成', this.widgets)
      this.dragWidgetId = ''
    },
    /* 拖拽到的内容区域 */
    widgetOnDraggedFn(e) {
      console.log('有元素拖拽到了内容区', e)
      // 获取结束坐标和列名
      const eventX = e.clientX // 结束在屏幕的x坐标
      const eventY = e.clientY // 结束在屏幕的y坐标

      // 获取工作台 dom 的 top&left 定位
      const workbenchPosition = this.getDomTopLeftById('workbench')
      const widgetTopInWorkbench = eventY - workbenchPosition.top - 25
      const widgetLeftInWorkbench = eventX - workbenchPosition.left - 50

      // 找出复制元素的 数据内容
      const obj = this.arr1.find(item => item.id === this.dragWidgetId)

      // 在工作台增加 一个新标签
      this.widgets.push({
        id: this.dragWidgetId,
        name: obj.name,
        value: {
          width: 100,
          height: 50,
          left: widgetLeftInWorkbench,
          top: widgetTopInWorkbench
        }
      })
    },
    dragOverFn(e) {
      console.log('正在拖拽')
      // 因为可能有多层dom，浏览器默认也不知道放哪层 https://blog.csdn.net/qq_41694291/article/details/101384209
      e.preventDefault()
      // https://blog.csdn.net/chaoyue1861/article/details/83926390
      e.stopPropagation()
      e.dataTransfer.dropEffect = 'copy'
    },
    // 获取dom在屏幕中的top和left
    getDomTopLeftById(id) {
      const dom = document.getElementById(id)
      let top = 0
      let left = 0
      if (dom != null) {
        top = dom.getBoundingClientRect().top
        left = dom.getBoundingClientRect().left
      }
      return { top: top, left: left }
    },
    // avue
    handleFocus({ index, left, top, width, height }) {
      console.log('聚焦了')
    },
    handleBlur({ index, left, top, width, height }) {
      console.log(143, index, left, top, width, height)

      this.$refs.draggableAvue[index].setActive(true)
      // 保存新增修改的组件
      if (left !== 0 && top !== 0) {
        this.widgets[index].value = {
          width,
          height,
          left,
          top
        }
      }
      // 判定如果 currentIndex 数组超过1个，则需要把上一个选中取消，只保留下一个选中
      if (this.currentIndex !== index && this.currentIndex !== '') {
        this.$refs.draggableAvue[this.currentIndex].setActive(false)
      }

      this.currentIndex = index
    },
    // 存储 widgets
    saveWidgetsFn() {
      localStorage.setItem('saveWidgetsFn', JSON.stringify(this.widgets))
      alert('存储成功！')
    },
    handleMousedown() {
      console.log('点击事件')
      if (this.currentIndex !== '') this.$refs.draggableAvue[this.currentIndex].setActive(false)
    }
  }
}
</script>

<style lang="scss" scoped>

@import './style/index.scss';

.workbench {
        background-color: #bdb9b9;
        position: relative;
        -webkit-user-select: none;
        -moz-user-select: none;
        -ms-user-select: none;
        user-select: none;
        -webkit-transform-origin: 0 0;
        transform-origin: 0 0;
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        position: absolute;
        list-style: none;

        .item2 {
      width: 100px;
      height: 50px;
      background-color: pink;
      cursor: all-scroll;
        }

        ::v-deep .avue-draggable {
          padding: 0;
        }
}

.vueRuler {

    ::v-deep .vue-ruler-content {
      width: 100%;
      height: 100%;
    }
}

</style>
