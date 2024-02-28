<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level<=2"
            type="text"
            size="mini"
            @click="() => treeAppend(data)">
            Append
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => treeEdit(data)">
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => treeRemove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>

      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="() => dialogSubmit()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
</script>

<script>
export default {
  components: {},
  props: {},
  data () {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: '',
      dialogType: '', // treeEdit or treeAppend
      category: {
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        productUnit: '',
        icon: ''
      },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: 'children',
        label: 'name'
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
          console.log('product category list 获得到菜单数据', data.data)
          this.menus = data.data
        } else {
          console.log('product category list 获得菜单数据失败', data)
        }
      })
    },
    // 批量删除
    batchDelete () {
      let catIds = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('被选中的元素', checkedNodes)
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
      }
      this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({ data }) => {
            this.$message({
              message: '菜单批量删除成功',
              type: 'success'
            })
            this.getMenus()
          })
        })
        .catch(() => {})
    },
    // 批量保存
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: '菜单顺序等修改成功',
          type: 'success'
        })
        // 刷新出新的菜单
        this.getMenus()
        // 设置需要默认展开的菜单
        this.expandedKey = this.pCid
        this.updateNodes = []
        this.maxLevel = 0
        // this.pCid = 0;
      })
    },
    handleDrop (draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)
      // 1、当前节点最新的父节点id
      let pCid = 0
      let siblings = null
      if (dropType === 'before' || dropType === 'after') {
        pCid =
          dropNode.parent.data.catId === undefined
            ? 0
            : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)

      // 2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level
          if (siblings[i].level !== draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level
            // 修改他子节点的层级
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          })
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
        }
      }

      // 3、当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)
    },
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          let cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      // 1、被拖动的当前节点以及所在的父节点总层数不能大于3

      // 1）、被拖动的当前节点总层数
      console.log('allowDrop:', draggingNode, dropNode, type)
      //
      this.countNodeLevel(draggingNode)
      // 当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      console.log('深度：', deep)

      //   this.maxLevel
      if (type === 'inner') {
        // console.log(
        //   `this.maxLevel：${this.maxLevel}；draggingNode.data.catLevel：${draggingNode.data.catLevel}；dropNode.level：${dropNode.level}`
        // );
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    countNodeLevel (node) {
      // 找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },
    treeRemove (node, data) {
      let ids = [data.catId]
      this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 发送 POST 请求给后端进行删除操作
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          console.log('removed ids success:', ids)
          // Step1. 获得所有菜单
          this.getMenus()
          // Step2. 设置需要展开的菜单
          this.expandedKey = [node.parent.data.catId]
        })

        this.$message({
          type: 'success',
          message: '删除成功!'
        })
      }).catch(() => {})
    },
    treeAppend (data) {
      this.title = '添加内容'
      this.dialogType = 'treeAppend'
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      // 清空历史数据
      this.category.name = ''
      this.category.catId = null
      this.category.icon = ''
      this.category.productUnit = ''
      this.category.sort = 0
      this.category.showStatus = 1
      console.log('Try call treeAppend', this.category)
      // 交给 Dialog 完成数据提交
      this.dialogVisible = true
    },
    treeEdit (data) {
      this.title = '修改分类'
      this.dialogType = 'treeEdit'
      // 发送请求获得当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({data}) => {
        // 更新需要回显的数据
        this.category.name = data.data.name
        this.category.catId = data.data.catId
        this.category.icon = data.data.icon
        this.category.productUnit = data.data.productUnit
        this.category.parentCid = data.data.parentCid
        this.category.catLevel = data.data.catLevel
        this.category.sort = data.data.sort
        this.category.showStatus = data.data.showStatus
        // 交给 Dialog 完成数据提交
        console.log('Try call treeEdit', this.category)
        this.dialogVisible = true
      })
    },
    dialogEditCategory () {
      // 发送 POST 请求, 修改三级分类数据
      let {catId, name, icon, productUnit} = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '修改成功!'
        })
        // 关闭对话框
        this.dialogVisible = false
        // Step1. 获得所有菜单
        this.getMenus()
        // Step2. 设置需要展开的菜单
        this.expandedKey = [this.category.parentCid]
      })
    },
    dialogAddCategory () {
      // 发送 POST 请求给后端进行删除操作
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        this.$message({
          type: 'success',
          message: '添加成功!'
        })
        // 关闭对话框
        this.dialogVisible = false
        // Step1. 获得所有菜单
        this.getMenus()
        // Step2. 设置需要展开的菜单
        this.expandedKey = [this.category.parentCid]
      })
    },
    dialogSubmit () {
      if (this.dialogType === 'treeEdit') {
        // 编辑逻辑
        this.dialogEditCategory()
      } else if (this.dialogType === 'treeAppend') {
        // 添加逻辑
        this.dialogAddCategory()
      }
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
