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
            @click="(e) => toggleExpand(!showExpand(item), item)"
          />
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
  },
  watch: {
    tree () {
      this.updateView()
    }
  },
  methods: {
    setCurrent (current) {
      if (current) {
        this.currentData = typeof current === 'object' ? current : { id: current }
      }
      this.updateView()
    },
    showExpand (node) {
      if (this.lazy) {
        return !node.isLeaf
      } else {
        return node.children && node.children.length
      }
    },
    nodeClick (node) {
      this.currentData = node
      this.$emit('node-click', node, node.parent)
    },
    updateView () {
      this.getContentHeight()
      this.handleScroll()
    },
    getContentHeight () {
      this.contentHeight = (this.flattenTree || []).filter(c => c.visible).length * this.option.itemHeight + 'px'
    },
    handleScroll () {
      const height = this.$refs.scroller.offsetHeight
      const scrollTop = this.$refs.scroller.scrollTop - height
      this.updateVisibleData(scrollTop > 0 ? scrollTop : 0)
    },
    updateVisibleData (scrollTop = 0) {
      const start = Math.floor(scrollTop / this.option.itemHeight)
      const end = start + this.visibleCount
      const allVisibleData = (this.flattenTree || []).filter(c => c.visible)
      this.visibleData = allVisibleData.slice(start, end)
      this.offset = start * this.option.itemHeight
    },
    toggleExpand (e, node) {
      if (e) return
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
    expand (node) {
      node.expand = true
      this.recursionVisible(node.expand, node.children, true)
      this.$emit('expand', node, node.parent)
    },
    collapse (node) {
      node.expand = false
      this.recursionVisible(node.expand, node.children, false)
      this.$emit('collapse', node, node.parent)
    },
    collapseAll (level = 1) {
      this.flattenTree.forEach(node => {
        node.expand = false
        if (node.level !== level) {
          node.visible = false
        }
      })
      this.updateView()
    },
    expandAll () {
      this.flattenTree.forEach(node => {
        node.expand = true
        node.visible = true
      })

      this.updateView()
    },
    recursionVisible (expand, children, status) {
      children.forEach(node => {
        node.visible = expand ? status : false
        if (node.children) {
          this.recursionVisible(node.expand, node.children, status)
        }
      })
    },
    filter (val) {
      (this.flattenTree || []).forEach(c => {
        c.visible = this.filterMethod(val, c)
      })
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
