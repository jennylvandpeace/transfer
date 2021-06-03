<template>
  <div class="el-transfer-panel">
    <p class="el-transfer-panel__header">
      <el-checkbox
        v-model="allChecked"
        :indeterminate="isIndeterminate"
        @change="handleAllCheckedChange"
      >
        {{ title }}
        <span>{{ checkedSummary }}</span>
      </el-checkbox>
    </p>

    <div :class="['el-transfer-panel__body', hasFooter ? 'is-with-footer' : '']">
      <el-input
        v-if="filterable"
        v-model="query"
        class="el-transfer-panel__filter"
        size="small"
        :placeholder="placeholder"
        @mouseenter.native="inputHover = true"
        @mouseleave.native="inputHover = false"
      >
        <i
          slot="prefix"
          :class="['el-input__icon', 'el-icon-' + inputIcon]"
          @click="clearQuery"
        />
      </el-input>
      <div v-if="panel === 'left'" class="panel">
        <el-checkbox-group
          v-show="!hasNoMatch && data.length > 0"
          v-model="checked"
          :class="{ 'is-filterable': filterable }"
          class="el-transfer-panel__list"
        >
          <el-checkbox
            v-for="item in filteredData"
            :key="item[keyProp]"
            class="el-transfer-panel__item"
            :label="item[keyProp]"
            :disabled="item[disabledProp]"
          >
            <option-content :option="item" />
            <span class="icon" @click="addValue(item[keyProp])">
              <i class="el-icon-arrow-right"  />
            </span>
          </el-checkbox>
        </el-checkbox-group>
      </div>
      <div v-if="panel === 'right'" class="panel">
        <el-checkbox-group
          v-show="!hasNoMatch && data.length > 0"
          v-model="checked"
          :class="{ 'is-filterable': filterable }"
          class="el-transfer-panel__list"
        >
          <el-checkbox
            v-for="item in filteredData"
            :key="item[keyProp]"
            :label="item[keyProp]"
            :class="item[disabledProp]? 'el-transfer-panel__item diff': 'el-transfer-panel__item'"
          >
            <option-content :option="item" />
            <span v-if="!item[disabledProp]" class="icon" @click="deleteValue(item[keyProp])">
              <i class="el-icon-close" />
            </span>
          </el-checkbox>
        </el-checkbox-group>
      </div>

      <p
        v-show="hasNoMatch"
        class="el-transfer-panel__empty"
      >{{ t('el.transfer.noMatch') }}</p>
      <p
        v-show="data.length === 0 && !hasNoMatch"
        class="el-transfer-panel__empty"
      >{{ t('el.transfer.noData') }}</p>
    </div>
    <p v-if="hasFooter" class="el-transfer-panel__footer">
      <slot />
    </p>
  </div>
</template>

<script>
import ElCheckboxGroup from 'element-ui/packages/checkbox-group'
import ElCheckbox from 'element-ui/packages/checkbox'
import ElInput from 'element-ui/packages/input'
import Locale from 'element-ui/src/mixins/locale'

export default {

  name: 'ElTransferPanel',

  components: {
    ElCheckboxGroup,
    ElCheckbox,
    ElInput,
    OptionContent: {
      props: {
        option: Object
      },
      render(h) {
        const getParent = vm => {
          if (vm.$options.componentName === 'ElTransferPanel') {
            return vm
          } else if (vm.$parent) {
            return getParent(vm.$parent)
          } else {
            return vm
          }
        }
        const panel = getParent(this)
        const transfer = panel.$parent || panel
        return <div style='height: 48px;text-align: left;color: #333;'>
          <span>{ this.option[panel.labelProp] } - { this.option[panel.functionProp] === 'property' ? '属性' : (this.option[panel.functionProp] === 'service' ? '服务' : '事件') }</span>
          <div style='line-height: 10px;color: #999999'>标识符：{ this.option[panel.keyProp] }</div>
        </div>
      }
    }
  },
  mixins: [Locale],

  componentName: 'ElTransferPanel',

  props: {
    data: {
      type: Array,
      default() {
        return []
      }
    },
    renderContent: Function,
    placeholder: String,
    title: String,
    filterable: Boolean,
    format: Object,
    filterMethod: Function,
    defaultChecked: Array,
    props: Object,
    panel: String
  },

  data() {
    return {
      checked: [],
      allChecked: false,
      query: '',
      inputHover: false,
      checkChangeByUser: true
    }
  },

  computed: {
    filteredData() {
      return this.data.filter(item => {
        if (typeof this.filterMethod === 'function') {
          return this.filterMethod(this.query, item)
        } else {
          const label = item[this.labelProp] || item[this.keyProp].toString()
          return label.toLowerCase().indexOf(this.query.toLowerCase()) > -1
        }
      })
    },

    checkableData() {
      return this.filteredData.filter(item => !item[this.disabledProp])
    },

    checkedSummary() {
      const checkedLength = this.checked.length
      const dataLength = this.data.length
      const { noChecked, hasChecked } = this.format
      if (noChecked && hasChecked) {
        return checkedLength > 0
          ? hasChecked.replace(/\${checked}/g, checkedLength).replace(/\${total}/g, dataLength)
          : noChecked.replace(/\${total}/g, dataLength)
      } else {
        return `${checkedLength}/${dataLength}`
      }
    },

    isIndeterminate() {
      const checkedLength = this.checked.length
      return checkedLength > 0 && checkedLength < this.checkableData.length
    },

    hasNoMatch() {
      return this.query.length > 0 && this.filteredData.length === 0
    },

    inputIcon() {
      return this.query.length > 0 && this.inputHover
        ? 'circle-close'
        : 'search'
    },

    labelProp() {
      return this.props.name || 'name'
    },

    functionProp() {
      return this.props.functionType || 'functionType'
    },

    keyProp() {
      return this.props.key || 'key'
    },

    disabledProp() {
      return this.props.isSelected || 'isSelected'
    },

    hasFooter() {
      return !!this.$slots.default
    }
  },

  watch: {
    checked(val, oldVal) {
      this.updateAllChecked()
      if (this.checkChangeByUser) {
        const movedKeys = val.concat(oldVal)
          .filter(v => val.indexOf(v) === -1 || oldVal.indexOf(v) === -1)
        this.$emit('checked-change', val, movedKeys)
      } else {
        this.$emit('checked-change', val)
        this.checkChangeByUser = true
      }
    },

    data() {
      const checked = []
      const filteredDataKeys = this.filteredData.map(item => item[this.keyProp])
      this.checked.forEach(item => {
        if (filteredDataKeys.indexOf(item) > -1) {
          checked.push(item)
        }
      })
      this.checkChangeByUser = false
      this.checked = checked
    },

    checkableData() {
      this.updateAllChecked()
    },

    defaultChecked: {
      immediate: true,
      handler(val, oldVal) {
        if (oldVal && val.length === oldVal.length &&
            val.every(item => oldVal.indexOf(item) > -1)) return
        const checked = []
        const checkableDataKeys = this.checkableData.map(item => item[this.keyProp])
        val.forEach(item => {
          if (checkableDataKeys.indexOf(item) > -1) {
            checked.push(item)
          }
        })
        this.checkChangeByUser = false
        this.checked = checked
      }
    }
  },

  methods: {
    clearChecked() {
      this.checked = []
    },
    addValue(key) {
      console.log(key)
      this.$emit('addValue', key)
    },
    deleteValue(key) {
      console.log(key)
      this.$emit('deleteValue', key)
    },
    updateAllChecked() {
      const checkableDataKeys = this.checkableData.map(item => item[this.keyProp])
      this.allChecked = checkableDataKeys.length > 0 &&
          checkableDataKeys.every(item => this.checked.indexOf(item) > -1)
    },

    handleAllCheckedChange(value) {
      this.checked = value
        ? this.checkableData.map(item => item[this.keyProp])
        : []
    },

    clearQuery() {
      if (this.inputIcon === 'circle-close') {
        this.query = ''
      }
    }
  }
}
</script>
<style lang="scss" scoped>
  .el-transfer-panel{
    width: 100%;
    border-radius: 0;
    ::v-deep .el-checkbox__inner{
      border-radius: 0;
    }
    .icon{
      position: absolute;
      right: 0;
      top:6px;
      width: 50px;
      text-align: right;
    }
    .panel{
      height: calc(100% - 47px);
    }
    ::v-deep .el-transfer-panel__list.is-filterable{
      padding-right: 15px;
      padding-left: 15px;
      height: 100%;
      overflow-x: hidden;
    }
    ::v-deep .el-transfer-panel__filter .el-input__inner{
      border-radius: 0;
    }
    .el-transfer-panel__item{
      width: 100%;
      position: relative;
      padding-right:0;
      padding-left:0;
      height: 56px;

      ::v-deep .el-checkbox {
        position: absolute;
        width: 100%;
        margin-right: 0;
      }
      ::v-deep .el-checkbox__input{
         left: 0;
         top: 16px;
      }
      &.diff{
        background: #F5F6FA;
        pointer-events:none;
        ::v-deep .el-checkbox__input{
          display: none;
        }
      }
    }
    ::v-deep .el-transfer-panel__item.el-checkbox .el-checkbox__label{
      position: relative;
    }
    .el-transfer-panel__header{
      text-align: left;
    }
    .el-transfer-panel__body{
      height:40vh;
    }
    .el-transfer-panel__footer {
      // text-align: left;
      .transfer-footer-left{
        position: absolute;
        left: 10px;
        top: 2px;
        font-size: 14px;
      }
      .transfer-footer-right{
        position: absolute;
        right: 10px;
        top: 2px;
        font-size: 14px;
      }
    }
  }

</style>
