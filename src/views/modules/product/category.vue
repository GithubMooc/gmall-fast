<template>
  <div>
    <el-switch v-model="draggable"
               active-text="开启拖拽"
               inactive-text="关闭拖拽"></el-switch>
    <el-button type="danger"
               @click="batchDelete">批量删除</el-button>
    <el-tree :data='menus'
             :props='defaultProps'
             :expand-on-click-node='expand'
             :show-checkbox='show'
             node-key="catId"
             :default-expanded-keys="expandedKeys"
             @node-expand="nodeExpand"
             @node-collapse="nodeCollapse"
             draggable
             :allow-drop="allowDrop"
             @node-drop="handleDrop"
             ref="menuTree">
      <span class="custom-tree-node"
            slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button type="text"
                     size="mini"
                     v-if="node.level<=2"
                     @click="() => append(data)">
            Append
          </el-button>
          <el-button type="text"
                     size="mini"
                     v-if="node.childNodes.length==0"
                     @click="() => remove(node, data)">
            Delete
          </el-button>
          <el-button type="text"
                     size="mini"
                     @click="() => edit(node, data)">
            Edit
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title"
               :visible.sync="dialogVisible"
               width="30%"
               :close-on-click-modal=false>
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name"
                    autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon"
                    autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"
                    autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="排序">
          <el-select v-model="category.sort"
                     placeholder="排序">
            <el-option v-for="item in options"
                       :key="item.value"
                       :label="item.label"
                       :value="item.value">
            </el-option>
          </el-select>
        </el-form-item>
      </el-form>
      <span slot="footer"
            class="dialog-footer">
        <el-button @click="cancel">取 消</el-button>
        <el-button type="primary"
                   @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
export default {
  data () {
    return {
      draggable: false,
      dialogType: '',
      title: '',
      category: {
        icon: '',
        productUnit: '',
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        oldName: ''
      },
      options: [{
        label: '零',
        value: 0
      },
      {
        label: '壹',
        value: 1
      },
      {
        label: '贰',
        value: 2
      },
      {
        label: '叁',
        value: 3
      },
      {
        label: '肆',
        value: 4
      },
      {
        label: '伍',
        value: 5
      },
      {
        label: '陆',
        value: 6
      },
      {
        label: '柒',
        value: 7
      },
      {
        label: '捌',
        value: 8
      },
      {
        label: '玖',
        value: 9
      },
      {
        label: '拾',
        value: 10
      }],
      dialogVisible: false,
      menus: [],
      expand: false,
      show: true,
      expandedKeys: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      },
      maxLevel: 0,
      updateNodes: []
    }
  },
  methods: {
    batchDelete () {
      let catIds = []
      let catNames = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      checkedNodes.forEach((item) => {
        catIds.push(item.catId)
        catNames.push(item.name)
      })
      this.$confirm(
        `是否删除【${catNames}】菜单？`,
        '提示',
        {
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
            if (data.msg === 'success') {
              this.$message({
                message: `菜单【${catNames}】删除成功！`, type: 'success'
              })
              this.getMenus()
            }
          })
        }).catch(() => {
          this.$message({
            message: `撤销删除【${catIds}】菜单！`, type: 'error'
          })
        })
    },
    handleDrop (draggingNode, dropNode, dropType) {
      let pCid = dropType === 'inner' ? dropNode.data.catId : (dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId)
      let siblings = dropType === 'inner' ? dropNode.childNodes : dropNode.parent.childNodes
      siblings.forEach((a, b) => {
        if (a.data.catId === draggingNode.data.catId) {
          let catLevel = draggingNode.catLevel
          if (a.level !== catLevel) {
            catLevel = a.level
            this.updateChildNodeLevel(a)
          }
          this.updateNodes.push({ catId: a.data.catId, sort: b, parentCid: pCid, catLevel: catLevel })
        } else {
          this.updateNodes.push({ catId: a.data.catId, sort: b })
        }
      })
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.dialogVisible = false
        if (data.msg === 'success') {
          this.getMenus()
          if (!this.expandedKeys.includes(pCid)) {
            this.expandedKeys.push(pCid)
          }
          this.$message({
            message: `菜单顺序修改成功！`,
            type: 'success'
          })
          this.updateNodes = []
          this.maxLevel = 0
        }
      })
    },
    updateChildNodeLevel (node) {
      if (node.childNodes.length > 0) {
        node.childNodes.forEach((item) => {
          this.updateNodes.push({ catId: item.data.catId, catLevel: item.level })
          this.updateChildNodeLevel(item)
        })
      }
    },
    allowDrop (draggingNode, dropNode, type) {
      if (!this.draggable) { return false }
      this.countNodeLevel(draggingNode.data)
      // 1、被拖动的当前节点以及所在的父节点总层数不能大于3
      let deep = this.maxLevel - draggingNode.data.catLevel + 1

      if (type === 'inner') {
        return (deep + dropNode.level) <= 3
      } else {
        return (deep + dropNode.parent.level) <= 3
      }
    },
    countNodeLevel (node) {
      this.maxLevel = node.catLevel
      if (node.children != null && node.children.length > 0) {
        node.children.forEach(element => {
          if (element.catLevel > this.maxLevel) {
            this.maxLevel = element.catLevel
          }
          this.countNodeLevel(element)
        })
      }
    },
    submitData () {
      if (this.dialogType === 'edit') {
        this.editCategory()
      }
      if (this.dialogType === 'add') {
        this.addCategory()
      }
    },
    cancel () {
      this.category.name = ''
      this.category.sort = 0
      this.dialogVisible = false
    },
    editCategory () {
      let { catId, name, icon, productUnit, oldName } = this.category
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({ catId, name, icon, productUnit }, false)
      }).then(({ data }) => {
        this.dialogVisible = false
        if (data.msg === 'success') {
          this.getMenus()
          if (!this.expandedKeys.includes(this.category.parentCid)) {
            this.expandedKeys.push(this.category.parentCid)
          }
          this.$message({
            message: `菜单【${oldName}】修改为【${name}】！`, type: 'success'
          })
        }
      })
    },
    edit (node, data) {
      this.dialogVisible = true
      this.dialogType = 'edit'
      this.title = '修改菜单'
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({ data }) => {
        this.category = data.data
        this.category.oldName = node.data.name
      })
    },
    addCategory () {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        if (data.msg === 'success') {
          this.$message({
            message: `菜单【${this.category.name}】保存成功！`, type: 'success'
          })
          this.dialogVisible = false
          if (!this.expandedKeys.includes(this.category.parentCid)) {
            this.expandedKeys.push(this.category.parentCid)
          }
          this.getMenus()
          this.category.name = ''
          this.category.sort = 0
        }
      })
    },
    getMenus () {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({ data }) => {
        this.menus = data.data
      })
    },
    nodeExpand (a) {
      this.expandedKeys.push(a.catId)
    },
    nodeCollapse (a) {
      let index = this.expandedKeys.indexOf(a.catId)
      this.expandedKeys.splice(index, 1)
    },
    append (data) {
      this.dialogType = 'add'
      this.title = '添加菜单'
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.sort = 0
      this.category.catId = null
      this.category.productUnit = ''
      this.category.icon = ''
      this.category.name = ''
      this.showStatus = 1
      this.dialogVisible = true
    },
    remove (node, data) {
      var ids = [data.catId]
      this.$confirm(
        `是否删除【${data.name}】菜单？`,
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            if (data.msg === 'success') {
              this.$message({
                message: `菜单【${node.data.name}】删除成功！`, type: 'success'
              })
              this.getMenus()
            }
          })
        }).catch(() => {
          this.$message({
            message: `撤销删除【${node.data.name}】菜单！`, type: 'error'
          })
        })
    }
  },
  created () {
    this.getMenus()
  }
}
</script>

<style>
</style>