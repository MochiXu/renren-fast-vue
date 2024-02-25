<template>
  <!--  <div>三级分类维护</div>-->
  <el-tree :data="data" :props="defaultProps" @node-click="handleNodeClick"></el-tree>
</template>
<script>
</script>

<script>
export default {
  components: {},
  props: {},
  data () {
    return {
      data: [],
      defaultProps: {
        children: 'children',
        label: 'label'
      }
    }
  },
  methods: {
    handleNodeClick (data) {
      console.log(data)
    },
    getMenus () {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        if (data && data.code === 0) {
          console.log('product category list 获得到菜单数据', data)
        } else {
          console.log('product category list 获得菜单数据失败', data)
        }
      })
    }
  },
  // 计算属性
  computed: {},
  // 监听方法
  watch: {},
  // 生命周期 - 创建完成(可以访问当前 this 实例)
  created () {
    this.getMenus()
  },
  // 生命周期 - 挂载完成(可以访问 DOM 元素)
  mounted () {},
  // 生命周期 - 创建之前
  beforeCreate () {},
  // 生命周期 - 挂载之前
  beforeMount () {},
  // 生命周期 - 更新之前
  beforeUpdate () {},
  // 生命周期 - 更新之后
  updated () {},
  // 生命周期 - 销毁之前
  beforeDestroy () {},
  // 生命周期 - 销毁完成
  destroyed () {},
  // 如果页面有 keep-alive 缓存功能, 这个函数会被触发
  activated () {}

}
</script>

<style scoped></style>
