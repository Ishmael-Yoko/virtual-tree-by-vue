<template>
  <div
    class="virtual-tree"
    ref="scroller"
    @scroll="handleScroll"
  >
    <div class="virtual-tree__phantom" :style="{ height: contentHeight }"></div>
    <div
      class="virtual-tree__content"
      :style="{ transform: `translateY(${offset}px)` }"
    >
      <template v-for="(item, index) in visibleData">
        <div
          :key="item.id"
          @click="nodeClick(item)"
          :class="{
            'virtual-tree-node': true,
            'is-current': item.id === currentData.id
          }"
          :style="{
            paddingLeft: 18 * (item.level - 1) + 'px',
          }"
        >
          <i
            :class="{
              'is_expand': item.expand,
              'el-icon-caret-right': true,
              'is_leaf': !showExpand(item)
            }"
            @click="(e) => toggleExpand(!showExpand(item), item, e)"
          />
          <el-checkbox
            v-if="showCheckbox"
            v-model="item.checked"
            :indeterminate="item.indeterminate"
            :disabled="!!item.disabled"
            @click.native.stop
            @change="(check) => handleCheckChange(check, item)"
          >
          </el-checkbox>
          <span
            v-if="item.loading"
            class="virtual-tree-node__loading-icon el-icon-loading">
          </span>
          <slot :item="item" :index="index"></slot>
        </div>
      </template>
    </div>
  </div>
</template>

<script>
// virtual-tree说明：
// virtual-tree是一个以虚拟列表为基础的tree组件，原理就是将树形结构数据扁平化为一个list，然后将list根据封装一个可视区域的list，通过滑动切换可视区域的数据。支持海量数据渲染
// 支持同步数据和异步加载数据
// 同步参数: tree: [{ id: 1, label: 'sss', children: []}]
// 异步参数: tree: [] ,load方法: 参数为当前节点数据，必须封装一个promise将要加入的数据resolve即可， lazy: 必须为true
// 支持过滤，传入filterMethod方法，第一个参数为当前值，第二个对象为当前节点对象
// 支持展开 闭合所有树
export default {
  name: 'virtual-tree',
  props: {
    tree: {
      type: Array,
      required: true,
      default: () => ([])
    },
    option: {
      type: Object,
      required: false,
      default: () => ({ itemHeight: 25 })
    },
    lazy: {
      type: Boolean,
      default: false
    },
    load: {
      type: Function
    },
    filterMethod: {
      type: Function
    },
    showCheckbox: {
      type: Boolean,
      default: false
    },
    expandKeys: {
      type: Array,
      default: () => ([])
    },
    checkedKeys: {
      type: Array,
      default: () => ([])
    }
  },
  data () {
    return {
      offset: 0,
      contentHeight: '0px',
      visibleData: [],
      currentData: {}
    }
  },
  computed: {
    flattenTree () {
      const flatten = function (list, childKey = 'children', level = 1, parent = null) {
        let arr = []
        list.forEach(item => {
          if (item.checked) {
            item.checked = true
          } else {
            item.checked = false
          }
          if (item.indeterminate) {
            item.indeterminate = true
          } else {
            item.indeterminate = false
          }
          item.disabled = !!item.disabled
          item.level = level
          if (item.visible === undefined) {
            item.visible = true
          }
          if (!parent.visible || !parent.expand) {
            item.visible = false
          }
          item.parent = parent
          arr.push(item)
          if (item[childKey]) {
            arr.push(
              ...flatten(
                item[childKey],
                childKey,
                level + 1,
                item
              )
            )
          }
        })
        return arr
      }
      return flatten(this.tree, 'children', 1, {
        level: 0,
        visible: true,
        expand: true,
        children: this.tree
      })
    },
    visibleCount () {
      const height = this.$refs.scroller.offsetHeight
      return Math.floor(height / this.option.itemHeight) * 3
    }
  },
  mounted () {
    this.updateView()
    this.$nextTick(() => {
      this.handleExpand()
    })
  },
  watch: {
    tree () {
      this.updateView()
      this.$nextTick(() => {
        this.handleExpand()
      })
    },
    expandKeys () {
      this.$nextTick(() => {
        this.handleExpand()
      })
    }
  },
  methods: {
    // 处理下拉
    handleExpand () {
      const filterNode = this.flattenTree.filter(c => this.expandKeys.find(k => k === c.id))
      filterNode.forEach(node => {
        this.toggleExpand(false, node, {})
      })
    },
    // 选中方法
    setCurrent (current) {
      if (current) {
        this.currentData = typeof current === 'object' ? current : { id: current }
      }
      this.updateView()
    },
    // 是否显示expand
    showExpand (node) {
      if (this.lazy) {
        return !node.isLeaf
      } else {
        return node.children && node.children.length
      }
    },
    // 点击节点的回调
    nodeClick (node) {
      this.currentData = node
      this.$emit('node-click', node, node.parent)
    },
    // 更新视图，包括更新内容区域的高度, 触发滑动事件
    updateView () {
      this.getContentHeight()
      this.handleScroll()
    },
    // 更新内容区域的高度
    getContentHeight () {
      this.contentHeight = (this.flattenTree || []).filter(c => c.visible).length * this.option.itemHeight + 'px'
    },
    // 滑动事件
    handleScroll () {
      const height = this.$refs.scroller.offsetHeight
      const scrollTop = this.$refs.scroller.scrollTop - height
      this.updateVisibleData(scrollTop > 0 ? scrollTop : 0)
    },
    // 更新可视区域的数据
    updateVisibleData (scrollTop = 0) {
      const start = Math.floor(scrollTop / this.option.itemHeight)
      const end = start + this.visibleCount
      const allVisibleData = (this.flattenTree || []).filter(c => c.visible)
      this.visibleData = allVisibleData.slice(start, end)
      this.offset = start * this.option.itemHeight
    },
    // 下拉展开或者合上
    toggleExpand (isE, node, e) {
      // e.stopPropagation()
      if (isE) return
      if (!this.lazy) {
        const isExpand = node.expand
        if (isExpand) {
          this.collapse(node, true)
        } else {
          this.expand(node, true)
        }
        this.updateView()
      } else {
        const isExpand = node.expand
        if (isExpand) {
          this.collapse(node, true)
          this.updateView()
        } else {
          if (node.children && node.children.length) {
            this.expand(node, true)
            this.updateView()
          } else {
            // 异步加载节点
            node.loading = true
            this.load(node).then(data => {
              node.loading = false
              node.children = data
              if (data.length === 0) {
                node.isLeaf = true
              }
              this.expand(node, true)
              this.updateView()
            }).catch(() => {
              node.loading = false
              node.children = []
              node.isLeaf = true
              this.expand(node, true)
              this.updateView()
            })
          }
        }
      }
    },

    // 展开节点
    expand (node) {
      node.expand = true
      this.recursionVisible(node.expand, node.children, true)
    },
    // 折叠节点
    collapse (node) {
      node.expand = false
      this.recursionVisible(node.expand, node.children, false)
    },

    // 折叠所有
    collapseAll (level = 1) {
      this.flattenTree.forEach(node => {
        node.expand = false
        if (node.level !== level) {
          node.visible = false
        }
      })
      this.updateView()
    },

    // 展开所有
    expandAll () {
      this.flattenTree.forEach(node => {
        node.expand = true
        node.visible = true
      })

      this.updateView()
    },

    // 递归节点
    recursionVisible (expand, children, status) {
      children.forEach(node => {
        node.visible = expand ? status : false
        if (node.children) {
          this.recursionVisible(node.expand, node.children, status)
        }
      })
    },
    handleCheckChange (check, node) {
      this.handleParentChange(check, node)
      this.handleChildrenChange(check, node)
      node.checked = check
      node.indeterminate = false
      this.updateView()
      this.$emit('check', {
        check: check,
        currentData: node,
        checked: this.getCheckedNodes(),
        halfChecked: this.getHalfCheckedNodes()
      })
    },
    handleParentChange (check, node) {
      const recursionCheck = (parent) => {
        const checkLength = parent.children.filter(node => !!node.checked)
        const allEmpty = parent.children.every(node => !node.checked)
        const allCheck = parent.children.every(node => !!node.checked)
        const otherCheck = checkLength.length > 0 && checkLength.length < parent.children.length
        const otherIndeterminate = parent.children.filter(node => !!node.indeterminate)
        if (allCheck) {
          parent.indeterminate = false
          parent.checked = allCheck
        }
        if (allEmpty) {
          parent.indeterminate = false
          parent.checked = !allEmpty
        }
        if (otherCheck || (otherIndeterminate.length > 0 && otherIndeterminate.length < parent.children.length)) {
          parent.checked = false
          parent.indeterminate = true
        }
        if (parent.parent && parent.id && parent.level !== 0 && parent.parent.children && parent.parent.children.length) {
          recursionCheck(parent.parent)
        }
      }
      if (node.parent && node.id && node.level !== 0 && node.parent.children && node.parent.children.length) {
        recursionCheck(node.parent)
      }
    },
    handleChildrenChange (check, node) {
      const recursionCheck = (childNodes) => {
        childNodes.forEach(node => {
          node.checked = check
          if (node.children && node.children.length) {
            recursionCheck(node.children)
          }
        })
      }
      if (node.children && node.children.length) {
        recursionCheck(node.children)
      }
    },
    getCheckedNodes () {
      return this.flattenTree.filter(c => c.checked)
    },
    getHalfCheckedNodes () {
      return this.flattenTree.filter(c => c.indeterminate)
    },
    // 添加过滤条件
    filter (val) {
      const recursionCheck = (arr, parent) => {
        arr.forEach(node => {
          node.visible = this.filterMethod(val, node)
          if (node.children && node.children.length) {
            recursionCheck(node.children, node)
          }
        })
        if (Object.keys(parent).length && parent.children.some(c => c.visible)) {
          parent.visible = true
        }
      }
      recursionCheck(this.tree, {})
      this.updateView()
    }
  }
}
</script>

<style lang='scss'>
.virtual-tree {
  position: relative;
  overflow-y: scroll;
  height: 100%;
}
.virtual-tree__phantom {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: -1;
}
.virtual-tree__content {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  min-height: 100px;
}
.virtual-tree-node {
  display: flex;
  cursor: pointer;
  height: 25px;
  line-height: 25px;
  color: #606266;
  box-sizing: border-box;
  .el-checkbox {
    margin-right: 8px!important;
  }
  &.is-current {
    background-color: #f2f7ff;
  }
  .virtual-tree-node__loading-icon {
    width: 15px;
    height: 15px;
    line-height: 15px;
    text-align: center;
    margin: 5px;
  }
  i {
    padding: 6px;
    cursor: pointer;
    color: #C0C4CC;
    font-size: 12px;
    transition: all 0.2s ease;
    &.is_expand {
      transform: rotateZ(90deg);
    }
    &.is_leaf {
      color: transparent;
      cursor: default;
    }
  }
  &:hover {
    background-color: #F5F7FA;
  }
}
.virtual-tree__content__item {
  padding: 5px;
  box-sizing: border-box;

  display: flex;
  justify-content: space-between;
  position: relative;
  align-items: center;
  cursor: pointer;
}
.virtual-tree__content__item:hover,
.virtual-tree__content__item__selected {
  background-color: #d7d7d7;
}
.virtual-tree__content__item__icon {
  position: absolute;
  left: 0;
  color: #c0c4cc;
  z-index: 10;
}
.virtual-tree__close {
  display: inline-block;
  width: 0;
  height: 0;
  overflow: hidden;
  font-size: 0;
  margin-right: 5px;
  border-width: 5px;
  border-color: transparent transparent transparent #c0c4cc;
  border-style: dashed dashed dashed solid;
}
.virtual-tree__expand {
  display: inline-block;
  width: 0;
  height: 0;
  overflow: hidden;
  font-size: 0;
  margin-right: 5px;
  border-width: 5px;
  border-color: #c0c4cc transparent transparent transparent;
  border-style: solid dashed dashed dashed;
}
</style>
