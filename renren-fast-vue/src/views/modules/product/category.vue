<template>
  <div>
    <el-switch v-model="draggable" active-text="Enable Draging" inactive-text="Disable Draging" style="margin-bottom: 15px"></el-switch>
    <el-button v-if="draggable" @click="batchSave">Batch Save</el-button>
    <el-button type="danger" @click="batchDelete">Batch Delete</el-button>
    <el-tree
      :expand-on-click-node="false"
      :data="menus"
      :props="defaultProps"
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
          <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">Append</el-button>
          <el-button type="text" size="mini" @click="edit(data)">Edit</el-button>
          <el-button v-if="node.childNodes.length === 0" type="text" size="mini" @click="() => remove(node, data)">Delete</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="Category Name">
          <el-input v-model="category.name" auto-complete="off" />
        </el-form-item>
        <el-form-item label="Icon">
          <el-input v-model="category.icon" auto-complete="off" />
        </el-form-item>
        <el-form-item label="Unit">
          <el-input v-model="category.productUnit" auto-complete="off" />
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button>Cancel</el-button>
        <el-button type="primary" @click="submitData">Confirm</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  // eslint-disable-next-line space-before-function-paren
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: '',
      dialogType: '',
      category: { name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, catId: null, productUnit: '', icon: '' },
      dialogVisible: false,
      expandedKey: [],
      menus: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  methods: {
    getMenus () {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({ data }) => {
        this.menus = data.data
      })
    },
    batchDelete () {
      let catIds = []
      let checkedNode = this.$refs.menuTree.getCheckedNodes()
      console.log(checkedNode)
      for (let i = 0; i < checkedNode.length; i++) {
        catIds.push(checkedNode[i].catId)
      }
      this.$confirm(`Confirm batch remove ${catIds} from category list`, 'Reminder', {
        confirmButtonText: 'Confirm',
        cancelButtonText: 'Cancel',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({ data }) => {
            this.$message({
              message: 'Menu delete successfully',
              type: 'success'
            })
            this.getMenus()
          })
        })
        .catch(() => {})
    },
    batchSave () {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(data => {
        this.$message({
          message: 'Update category order successfully',
          type: 'success'
        })
        this.getMenus()
        this.expandedKey = this.pCid
        this.updateNodes = []
        this.maxLevel = 0
        // this.pCid = 0
      })
    },

    handleDrop (draggingNode, dropNode, dropType, ev) {
      // 1. 当前拖拽节点的最新父节点id
      // console.log(draggingNode, dropNode, dropType, ev)
      let pCid = 0
      let siblings = null
      if (dropType === 'before' || dropType === 'after') {
        pCid = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)
      // 2. 当前拖拽节点的最新排序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // current draging node
          let currentCatLevel = draggingNode.level
          if (siblings[i].level !== draggingNode.level) {
            // catLevel change
            currentCatLevel = siblings[i].level
            // update childNode catLevel
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: currentCatLevel })
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
        }
      }

      // 3. 当前拖拽节点的最新层级
    },
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          let cNode = node.childNodes[i].data
          this.updateNodes.push({ catId: cNode.catId, catLevel: node.childNodes[i].level })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      // 被拖动的当前节点以及所在的父节点总层数不能大于3
      // 1. 判断被拖动的当前节点总层数
      this.countCurrentNodeLevel(draggingNode)
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      // 当前正在拖动的节点 + 父节点所在的深度不大于3即可
      if (type === 'inner') {
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    countCurrentNodeLevel (node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countCurrentNodeLevel(node.childNodes[i])
        }
      }
    },

    append (data) {
      this.dialogType = 'add'
      this.title = 'New category'
      this.dialogVisible = true

      this.category = {
        catId: null,
        icon: '',
        productUnit: '',
        sort: 0,
        showStatus: 1
      }
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
    },
    edit (data) {
      console.log(data)
      this.dialogType = 'edit'
      this.title = 'Edit category'
      this.dialogVisible = true
      // send request to get current category
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(data => {
        this.category.name = data.data.category.name
        this.category.catId = data.data.category.catId
        this.category.icon = data.data.category.icon
        this.category.productUnit = data.data.category.productUnit
        this.category.parentCid = data.data.category.parentCid
        // this.category.catLevel = data.data.category.catLevel
        // this.category.sort = data.data.category.sort
        // this.category.showStatus = data.data.category.showStatus
      })
    },
    remove (node, data) {
      let ids = [data.catId]
      this.$confirm(`Confirm remove ${data.name} from category list`, 'Reminder', {
        confirmButtonText: 'Confirm',
        cancelButtonText: 'Cancel',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            this.$message({
              type: 'success',
              message: 'Delete success',
              duration: 2000
            })
            this.getMenus()
            this.expandedKey = [node.parent.data.catId]
          })
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: 'Delete Failed',
            duration: 2000
          })
        })
    },
    submitData () {
      if (this.dialogType === 'add') {
        this.addCategory()
      }
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
    },
    editCategory () {
      let { name, catId, icon, productUnit } = this.category
      this.$http({
        url: this.$http.adornUrl(`/product/category/update`),
        method: 'post',
        data: this.$http.adornData({ name, catId, icon, productUnit }, false)
      }).then(data => {
        this.$message({
          type: 'success',
          message: 'Update success',
          duration: 2000
        })
        this.dialogVisible = false
        this.getMenus()
        this.expandedKey = [this.category.parentCid]
      })
    },
    addCategory () {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.dialogVisible = false

        this.$message({
          type: 'success',
          message: 'Create success',
          duration: 2000
        })

        this.getMenus()
        console.log(this.category.parentCid)
        this.expandedKey = [this.category.parentCid]
      })
    }
  },
  created () {
    this.getMenus()
  }
}
</script>
<style lang="scss" scoped>
