<template>
  <!-- 滚动的盒子 -->
  <div class="viewport"
       ref="viewport"
       @scroll="scrollFn">
    <!-- 自定义滚动条 -->
    <div class="scroll-bar"
         ref="scrollBar"></div>
    <!-- 真实渲染的内容 vid是一个计数的标识 方便看当前是第几个-->
    <div class="scroll-list">
      <div v-for="item in visibleData"
           :key="item.id"
           :vid="item.id"
           ref="items"
           :style="{transform:`translate3d(0,${offset}px,0)`}">
        <slot :item="item"></slot>
      </div>
    </div>
  </div>
</template>

<script>
import { throttle } from 'lodash';
export default {
  props: {
    size: Number,
    remain: Number,
    items: Array,
    variable: Boolean,
  },
  data () {
    return {
      start: 0,
      end: this.remain, // 默认显示8条
      offset: 0 // 偏移量
    }
  },
  created () {
    // leading为false 表示第一次不触发节点函数
    this.scrollFn = throttle(this.handleScroll, 200, { leading: false });
  },
  computed: {
    // 为了防止用户在快速滑动的过程中,上下区域出现空白,可以上下预先加载一屏数据
    // 预留上一屏数据
    prevCount () { // 前面预留几个
      return Math.min(this.start, this.remain);
    },
    // 预留下面一屏数据
    nextCount () { // 后面预留几个
      return Math.min(this.remain, this.items.length - this.end);
    },
    // 可见数据
    visibleData () {
      // return this.items.slice(this.start, this.end);
      // 一次截取三屏的数据
      let start = this.start - this.prevCount;
      let end = this.end + this.nextCount;
      return this.items.slice(start, end);
    }
  },
  mounted () {
    // 可视区域高度
    this.$refs.viewport.style.height = this.size * this.remain + 'px';
    // 滚动条高度
    this.$refs.scrollBar.style.height = this.items.length * this.size + 'px';

    // 如果加载完毕 需要缓存每一项的高度
    this.cacheList();
    // 1.先记录好,等一会滚动的时候,去渲染页面时获取真实dom的高度,来替换掉缓存的内容

    // 2.重新计算滚动条高度
  },
  updated () {
    // 页面渲染完成后,需要根据当前展示的数据,更新缓存区的内容
    // 在updated钩子里也拿不到更新后的真实dom
    this.$nextTick(() => {
      // 根据当前显示的数据,更新缓存中的height bottom top,最终更新滚动条的高度
      // 获取item的节点
      let nodes = this.$refs.items;
      // 当前item不存在就return
      if (!(nodes && nodes.length > 0)) return;
      nodes.forEach(node => {
        let { height } = node.getBoundingClientRect(); // 真实的item高度
        // 更新缓存的高度
        let id = node.getAttribute('vid') - 0; // 减0转换成number类型
        // 获取缓存的高度
        let oldHeight = this.positions[id].height;
        // 计算当前的高度和之前缓存的高度是否有变化
        let val = oldHeight - height;
        if (val) {
          // 如果缓存的高度和真实的高度一样,就不用做任何操作
          this.positions[id].height = height;
          this.positions[id].bottom = this.positions[id].bottom - val; // 底部增加了
          // 后续的所有项都要向后移动 类似链表
          for (let i = id + 1; i < this.positions.length; i++) {
            this.positions[i].top = this.positions[i - 1].bottom;
            this.positions[i].bottom = this.positions[i].bottom - val;
          }
        }
      });
      // 只要更新过就算出滚动条的最新高度
      // 就是动态的计算滚动条的高度
      this.$refs.scrollBar.style.height = this.positions[this.positions.length - 1].bottom + 'px';
    })
  },
  methods: {
    cacheList () { // 缓存当前项的高度和top值 还有bottom
      this.positions = this.items.map((item, index) => ({
        height: this.size,
        top: index * this.size, // 当前个数乘以size
        bottom: (index + 1) * this.size // 当前项的bottom和top值差一个高度,即当前项的bottom就是下一项的top值
      }));
      // console.log(this.positions);
    },
    getStartIndex (value) { // 查找当前滚动条需要找到的值
      let start = 0; // 开始位置
      let end = this.positions.length - 1; // 结束位置
      let temp = null; // 标记值,用来保存上一轮的middleIndex
      while (start <= end) {
        let middleIndex = parseInt((start + end) / 2);
        // console.log(middleIndex,'index');

        let middleValue = this.positions[middleIndex].bottom; // 找到当前列表中间那个值的结尾点
        // console.log(middleIndex,'index');

        if (middleValue === value) { // 如果直接找到了 就返回当前项的下一项 下一项才是要显示的值
          return middleIndex + 1;
        } else if (middleValue < value) { // 当前要查找的那一项value 在右边
          // 把start指针 移到中间值middleValue的前面 左边区域和中间值都不是我们要找的值
          start = middleIndex + 1;
        } else if (middleValue > value) { // 当前要查找的那一项value 在左边
          // 原始的二分查找,找不到目标值会return -1 ,这里稍微做了调整
          if (temp == null || temp > middleIndex) {
            temp = middleIndex; // 找到目标值的范围
          }
          // 把end指针 移到中间值middleValue的前面
          end = middleIndex - 1;
        }
      }
      return temp;
    },
    handleScroll () {
      // 1.应该先算出来当前滚过去了几个 应该从第几个开始显示
      // 当前滚过去的总距离
      let scrollTop = this.$refs.viewport.scrollTop;
      if (this.variable) {
        // 如果有设置variable，说明当前item高度未知 则需要使用二分查找 找到对应的记录
        // 二分查找的思想：假设要在有序列表查找a,先把数组从中间一分为二，如果右边的数组最小值比a大，
        // 那就在左边的数组中找，思路和前面一样，就是不断缩小范围，直到找到目标值

        // 先获取到开始位置
        this.start = this.getStartIndex(scrollTop);
        // 计算结束位置
        this.end = this.start + this.remain;
        // 设置偏移量 减去预渲染的item
        this.offset = this.positions[this.start - this.prevCount]
          ? this.positions[this.start - this.prevCount].top
          : 0;
      } else {
        // 2.获取当前应该从第几个开始渲染,滚动的总距离除以每个item的高度就是当前要显示的那一个item
        this.start = Math.floor(scrollTop / this.size);
        // 3.计算当前结尾的位置,可视区域默认显示8条数据
        this.end = this.start + this.remain;
        // 定位当前的可视区域 让当前渲染的内容在当前的viewport可视区域内
        // this.offset = this.start * this.size; // 调整偏移量
        // 如果有预留渲染数据,应该把当前位置偏移量向上移动这么多,即所有预留数据item总高度
        this.offset = this.start * this.size - this.size * this.prevCount;
      }
    }
  }
}
</script>

<style lang="stylus">
.viewport
  overflow-y scroll
  position relative
  .scroll-list
    position absolute
    left 0
    top 0
    width 100%
</style>