<template>
  <el-row class="transfer-user">
    <el-col :span="12">
      <transfer-panel
        ref="leftPanel"
        v-bind="$props"
        :panel="leftPanel"
        :data="sourceData"
        :title="titles[0] || t('el.transfer.titles.0')"
        :default-checked="leftDefaultChecked"
        :placeholder="filterPlaceholder || t('el.transfer.filterPlaceholder')"
        @checked-change="onSourceCheckedChange"
        @addValue="addValue"
        @deleteValue="deleteValue"
      >
        <slot name="left-footer" />
      </transfer-panel>
    </el-col>
    <el-col :span="12">
      <transfer-panel
        ref="rightPanel"
        :panel="rightPanel"
        v-bind="$props"
        :data="targetData"
        :title="titles[1] || t('el.transfer.titles.1')"
        :default-checked="rightDefaultChecked"
        :placeholder="filterPlaceholder || t('el.transfer.filterPlaceholder')"
        @checked-change="onTargetCheckedChange"
        @addValue="addValue"
        @deleteValue="deleteValue"
      >
        <slot name="right-footer" />
      </transfer-panel>
    </el-col>
  </el-row>
</template>

<script>
import Emitter from 'element-ui/src/mixins/emitter'
import Locale from 'element-ui/src/mixins/locale'
import TransferPanel from './transfer-panel.vue'
import Migrating from 'element-ui/src/mixins/migrating'

export default {
  name: 'Transfer',

  components: {
    TransferPanel
  },

  mixins: [Emitter, Locale, Migrating],

  props: {
    data: {
      type: Array,
      default() {
        return []
      }
    },
    titles: {
      type: Array,
      default() {
        return []
      }
    },
    buttonTexts: {
      type: Array,
      default() {
        return []
      }
    },
    filterPlaceholder: {
      type: String,
      default: ''
    },
    filterMethod: Function,
    leftDefaultChecked: {
      type: Array,
      default() {
        return []
      }
    },
    rightDefaultChecked: {
      type: Array,
      default() {
        return []
      }
    },
    renderContent: Function,
    value: {
      type: Array,
      default() {
        return []
      }
    },
    format: {
      type: Object,
      default() {
        return {}
      }
    },
    filterable: Boolean,
    props: {
      type: Object,
      default() {
        return {
          name: 'name',
          functionType: 'functionType',
          key: 'key',
          isSelected: 'isSelected'
        }
      }
    },
    targetOrder: {
      type: String,
      default: 'original'
    }
  },

  data() {
    return {
      leftPanel: 'left',
      rightPanel: 'right',
      leftChecked: [],
      rightChecked: []
    }
  },

  computed: {
    dataObj() {
      const key = this.props.key
      return this.data.reduce((o, cur) => (o[cur[key]] = cur) && o, {})
    },

    sourceData() {
      return this.data.filter(item => this.value.indexOf(item[this.props.key]) === -1)
    },

    targetData() {
      if (this.targetOrder === 'original') {
        const arr = this.data.filter(item => this.value.indexOf(item[this.props.key]) > -1)
        return arr.sort(function(a, b) { return (a.isSelected < b.isSelected) ? -1 : 1 })
      } else {
        return this.value.reduce((arr, cur) => {
          const val = this.dataObj[cur]
          if (val) {
            arr.push(val)
          }
          arr.sort(function(a, b) {
            return (a.isSelected < b.isSelected) ? -1 : 1
          })
          return arr
        }, [])
      }
    },

    hasButtonTexts() {
      return this.buttonTexts.length === 2
    }
  },

  watch: {
    value(val) {
      this.dispatch('ElFormItem', 'el.form.change', val)
    }
  },

  methods: {
    addValue(key) {
      this.$emit('addValue', key)
    },
    deleteValue(key) {
      this.$emit('deleteValue', key)
    },
    getMigratingConfig() {
      return {
        props: {
          'footer-format': 'footer-format is renamed to format.'
        }
      }
    },

    onSourceCheckedChange(val, movedKeys) {
      this.leftChecked = val
      if (movedKeys === undefined) return
      this.$emit('left-check-change', val, movedKeys)
    },

    onTargetCheckedChange(val, movedKeys) {
      this.rightChecked = val
      if (movedKeys === undefined) return
      this.$emit('right-check-change', val, movedKeys)
    },

    addToLeft() {
      const currentValue = this.value.slice()
      this.rightChecked.forEach(item => {
        const index = currentValue.indexOf(item)
        if (index > -1) {
          currentValue.splice(index, 1)
        }
      })
      this.$emit('input', currentValue)
      this.$emit('change', currentValue, 'left', this.rightChecked)
    },

    addToRight() {
      let currentValue = this.value.slice()
      const itemsToBeMoved = []
      const key = this.props.key
      this.data.forEach(item => {
        const itemKey = item[key]
        if (
          this.leftChecked.indexOf(itemKey) > -1 &&
            this.value.indexOf(itemKey) === -1
        ) {
          itemsToBeMoved.push(itemKey)
        }
      })
      currentValue = this.targetOrder === 'unshift'
        ? itemsToBeMoved.concat(currentValue)
        : currentValue.concat(itemsToBeMoved)
      this.$emit('input', currentValue)
      this.$emit('change', currentValue, 'right', this.leftChecked)
    },

    clearQuery(which) {
      if (which === 'left') {
        this.$refs.leftPanel.query = ''
      } else if (which === 'right') {
        this.$refs.rightPanel.query = ''
      }
    }
  }
}
</script>
<style lang="scss" scoped>
  .transfer-user{
    width: 100%;
    text-align: center;
  }
</style>
