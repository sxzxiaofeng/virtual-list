<template>
 <!-- <div style="height:40px;border:1px solid #64f> -->
   <!-- 不设置高度,item高度未知 不确定的情况 高度由内容撑开-->
 <div style="border:1px solid #64f;padding:10px 0">
   {{ item.value }}
 </div>
</template>

<script>
export default {
  props:{
    item:Object
  }
}
</script>