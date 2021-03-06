<template>
  <div class="q-data-table shadow-1">
    <div v-if="hasToolbar && toolbar === ''" class="q-data-table-toolbar upper-toolbar row reverse-wrap items-center justify-end">
      <div v-if="config.title" class="q-data-table-title ellipsis auto" v-html="config.title"></div>
      <div class="row items-center">
        <button v-if="config.refresh && !refreshing" class="primary clear" @click="refresh">
          <i>refresh</i>
        </button>
        <button v-if="refreshing" class="disabled">
          <i class="animate-spin-reverse">cached</i>
        </button>
        <button v-if="config.filter" class="primary clear" @click="toolbar = 'filter'">
          <i>filter_list</i>
        </button>
        <q-select
          type="toggle"
          v-if="config.columnPicker"
          v-model="columnSelection"
          :options="columnSelectionOptions"
          static-label="Columns"
          class="text-right"
          style="margin-left: 10px"
        ></q-select>
      </div>
    </div>

    <table-filter v-if="toolbar === 'filter'" :filtering="filtering" :columns="cols" @close="toolbar = ''"></table-filter>
    <div class="q-data-table-toolbar upper-toolbar row reverse-wrap items-center justify-end q-data-table-selection" v-show="toolbar === 'selection'">
      <div class="auto">
        {{ rowsSelected }} item<span v-show="rowsSelected > 1">s</span> selected.
        <button class="primary clear small" @click="clearSelection()">Clear</button>
      </div>
      <div>
        <slot name="selection" :rows="selectedRows"></slot>
      </div>
    </div>

    <template v-if="responsive">
      <div v-if="message" class="q-data-table-message row items-center justify-center" v-html="message"></div>
      <table v-else class="q-table horizontal-delimiter responsive" ref="body">
        <tbody>
          <tr v-for="(row, index) in rows">
            <td v-if="config.selection">
              <q-checkbox v-if="config.selection === 'multiple'" v-model="rowSelection[index]"></q-checkbox>
              <q-radio v-else v-model="rowSelection[0]" :val="index"></q-radio>
            </td>
            <td v-for="col in cols" :data-th="col.label" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
              <span v-if="!$scopedSlots['col-'+col.field]" v-html="format(row, col)"></span>
              <slot v-if="$scopedSlots['col-'+col.field]" :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]"></slot>
            </td>
          </tr>
        </tbody>
      </table>
    </template>

    <div v-else class="q-data-table-container" @mousewheel="mouseWheel" @DOMMouseScroll="mouseWheel">
      <div class="q-data-table-head" ref="head" :style="{marginRight: scroll.vert}">
        <table-content head :cols="cols" :sorting="sorting" :scroll="scroll" :selection="config.selection" @sort="setSortField"></table-content>
      </div>
      <div
        class="q-data-table-body"
        :style="bodyStyle"
        ref="body"
        @scroll="scrollHandler"
      >
        <div v-if="message" class="q-data-table-message row items-center justify-center" v-html="message"></div>
        <table-content v-else :cols="cols" :selection="config.selection">
          <tr v-for="row in rows" :style="rowStyle">
            <td v-if="config.selection"></td>
            <td v-if="leftStickyColumns" :colspan="leftStickyColumns"></td>
            <td v-for="col in regularCols" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
              <span v-if="!$scopedSlots['col-'+col.field]" v-html="format(row, col)"></span>
              <slot v-if="$scopedSlots['col-'+col.field]" :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]"></slot>
            </td>
            <td v-if="rightStickyColumns" :colspan="rightStickyColumns"></td>
          </tr>
        </table-content>
      </div>

      <template v-if="!message && (leftStickyColumns || config.selection)">
        <div
          class="q-data-table-sticky-left"
          ref="stickyLeft"
          :style="{bottom: scroll.horiz}"
        >
          <table-sticky :sticky-cols="leftStickyColumns" :cols="cols" :sorting="sorting" :selection="config.selection">
            <tr v-for="(row, index) in rows" :style="rowStyle">
              <td v-if="config.selection">
                <q-checkbox v-if="config.selection === 'multiple'" v-model="rowSelection[index]"></q-checkbox>
                <q-radio v-else v-model="rowSelection[0]" :val="index"></q-radio>
              </td>
              <td v-for="col in leftCols" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
                <span v-if="!$scopedSlots['col-'+col.field]" v-html="format(row, col)"></span>
                <slot v-if="$scopedSlots['col-'+col.field]" :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]"></slot>
              </td>
            </tr>
          </table-sticky>
        </div>
        <div class="q-data-table-sticky-left" :style="{bottom: scroll.horiz}">
          <table-sticky head :sticky-cols="leftStickyColumns" :scroll="scroll" :cols="cols" :sorting="sorting" @sort="setSortField" :selection="config.selection"></table-sticky>
        </div>
      </template>

      <template v-if="!message && rightStickyColumns">
        <div
          class="q-data-table-sticky-right"
          ref="stickyRight"
          :style="{right: scroll.vert, bottom: scroll.horiz}"
        >
          <table-sticky right :sticky-cols="rightStickyColumns" :cols="cols" :sorting="sorting" :selection="config.selection">
            <tr v-for="row in rows" :style="rowStyle">
              <td v-if="config.selection" class="invisible"></td>
              <td :colspan="cols.length - rightStickyColumns" class="invisible"></td>
              <td v-for="col in rightCols" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
                <span v-if="!$scopedSlots['col-'+col.field]" v-html="format(row, col)"></span>
                <slot v-if="$scopedSlots['col-'+col.field]" :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]"></slot>
              </td>
            </tr>
          </table-sticky>
        </div>
        <div class="q-data-table-sticky-right" :style="{right: scroll.vert}">
          <table-sticky right head :sticky-cols="rightStickyColumns" :scroll="scroll" :cols="cols" :sorting="sorting" @sort="setSortField" :selection="config.selection"></table-sticky>
        </div>
      </template>
    </div>

    <table-pagination v-if="config.pagination" :pagination="pagination" :entries="pagination.entries"></table-pagination>
  </div>
</template>

<script>
import Utils from '../../utils'

import ColumnSelection from './plugins/column-selection/column-selection'
import Filter from './plugins/filter/filter'
import Pagination from './plugins/pagination/pagination'
import Responsive from './plugins/responsive/responsive'
import RowSelection from './plugins/row-selection/row-selection'
import Scroll from './plugins/scroll/scroll'
import Sort from './plugins/sort/sort'
import StickyColumns from './plugins/sticky-cols/sticky-cols'

import TableContent from './TableContent.vue'

export default {
  mixins: [ColumnSelection, Filter, Pagination, Responsive, RowSelection, Scroll, Sort, StickyColumns],
  props: {
    data: {
      type: Array,
      default: []
    },
    columns: {
      type: Array,
      required: true
    },
    config: {
      type: Object,
      default () { return {} }
    }
  },
  data () {
    return {
      selected: false,
      toolbar: '',
      refreshing: false
    }
  },
  computed: {
    rows () {
      let rows = Utils.clone(this.data)

      rows.forEach((row, i) => {
        row.__index = i
      })

      if (this.filtering.terms) {
        rows = this.filter(rows)
      }

      if (this.sorting.field) {
        this.sort(rows)
      }

      this.pagination.entries = rows.length

      if (this.pagination.rowsPerPage > 0) {
        rows = this.paginate(rows)
      }

      return rows
    },
    rowStyle () {
      if (this.config.rowHeight) {
        return {height: this.config.rowHeight}
      }
    },
    bodyStyle () {
      return this.config.bodyStyle || {}
    },
    message () {
      if (this.rows.length) {
        return false
      }

      if (this.filtering.terms) {
        return this.config.messages.noDataAfterFiltering || '<i>warning</i> No results. Please refine your search terms.'
      }

      return this.config.message.noData || '<i>warning</i> No data available to show.'
    },
    hasToolbar () {
      return this.config.title || this.config.filter || this.config.columnPicker || this.config.refresh
    }
  },
  methods: {
    resetBody () {
      let body = this.$refs.body

      if (body) {
        body.scrollTop = 0
      }
      this.pagination.page = 1
    },
    format (row, col) {
      return col.format ? col.format(row[col.field]) : row[col.field]
    },
    refresh (state) {
      if (state === false) {
        this.refreshing = false
      }
      else if (state === true || !this.refreshing) {
        this.refreshing = true
        this.$emit('refresh', () => {
          this.refreshing = false
        })
      }
    },
    formatStyle (col, value) {
      return typeof col.style === 'function' ? col.style(value) : col.style
    },
    formatClass (col, value) {
      return typeof col.classes === 'function' ? col.classes(value) : col.classes
    }
  },
  components: {
    TableContent
  }
}
</script>
