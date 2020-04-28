<template>
  <div id="app">
    <!-- 只显示可视区域 -->
    <!-- 告诉列表的每一项多高，计算出滚动条 -->
    <!-- size是每一项的高度 remain表示可视区域显示8条数据 -->
    <!-- variable 表示当前item的高度不确定 当高度不确定时,size只是假设的高度-->
    <VirtualList :size="40"
                 :remain="8"
                 :items="items"
                 :variable="true"
                 >
                 <!-- 作用域插槽 
                  在父组件传参，子组件调用执行
                 -->
      <Item slot-scope="{item}" :item="item"></Item>             
    </VirtualList>
  </div>
</template>

<script>
import VirtualList from './components/VirtualList';
import Item from './components/item';
import Mock from 'mockjs';
let items = [];
for (let i = 0; i < 10000; i++) {
  // 用mock随机造点英文数据 字符长度在5-50之间 sentence是造句的意思
  items.push({ id: i, value: Mock.Random.sentence(5,50)});
}
export default {
  name: 'App',
  components: {
    VirtualList,
    Item
  },
  data () {
    return { items }
  }
};
</script>

<style lang="stylus">
* 
  margin 0 
  padding 0
  box-sizing border-box
</style>
