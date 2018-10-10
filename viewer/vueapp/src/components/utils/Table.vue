<template>

  <!-- TODO fit table -->
  <!-- TODO animations? -->
  <table v-if="computedColumns && computedColumns.length"
    style="{ width: tableWidth + 'px' }"
    class="table-striped"
    :class="tableClasses"
    :id="id">
    <thead>
      <tr ref="draggableColumns">
        <!-- TODO col config buttons -->
        <th v-if="actionColumn"
          class="ignore-element">
          &nbsp;
        </th>
        <!-- TODO save table state on sort -->
        <th v-for="column in computedColumns"
          :key="column.name"
          v-b-tooltip.hover
          :title="column.help"
          @click="sort(column.sort)"
          :style="{'width': column.width + 'px'}"
          :class="{'cursor-pointer':column.sort}">
          {{ column.name }}
          <span v-if="column.sort">
            <span v-show="tableSortField === column.sort && !tableDesc" class="fa fa-sort-asc"></span>
            <span v-show="tableSortField === column.sort && tableDesc" class="fa fa-sort-desc"></span>
            <span v-show="tableSortField !== column.sort" class="fa fa-sort"></span>
          </span>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="item of data"
        :key="item.id">
        <!-- TODO action buttons -->
        <td v-if="actionColumn">
          &nbsp;
        </td>
        <td v-for="(column, index) in computedColumns"
          :key="column.dataField+index">
          <template v-if="!column.dataFunction">
            {{ item[column.dataField] }}
          </template>
          <template v-else>
            {{ column.dataFunction(item[column.dataField]) }}
          </template>
        </td>
      </tr>
      <tr v-if="noResults && data && !data.length">
        <td colspan="6"
          class="text-danger text-center">
          <span class="fa fa-warning">
          </span>&nbsp;
          No results match your search
        </td>
      </tr>
    </tbody>
  </table>

</template>

<script>
import UserService from '../users/UserService';

import Sortable from 'sortablejs';
import '../../../../public/colResizable.js';

let tableDestroyed;
let draggableColumns;

/**
 * IMPORTANT! This component kicks off the loading of the
 * data in the table once the table state has been loaded.
 * There is no need to load data in the parent.
 */
export default {
  name: 'MolochTable',
  props: {
    loadData: { // event to fire when the table needs to load data
      type: Function,
      required: true
    },
    id: { // unique id of the table
      type: String,
      required: false
    },
    tableClasses: { // table classes to be applied to the table
      type: String,
      require: false
    },
    columns: { // columns to be displayed in the table
      type: Array,
      required: true
    },
    actionColumn: { // whether to display an action column on the left
      type: Boolean,
      default: false
    },
    data: { // table data
      type: Array
    },
    noResults: { // whether or not to dispaly a no results row if data array is empty
      type: Boolean,
      default: false
    },
    sortField: { // the field that the query is sorting by
      type: String,
      required: true
    },
    desc: { // the direction the query is sorting by
      type: Boolean,
      required: true
    },
    tableStateName: { // api endpoint to save table state (/state/:tableStateName)
      type: String,
      required: true
    },
    tableWidthsStateName: { // api endpoint to save table state (/state/:tableWidthsStateName)
      type: String,
      required: true
    }
  },
  data: function () {
    return {
      error: '',
      tableDesc: undefined,
      tableSortField: undefined,
      computedColumns: [], // columns in the order computed from the saved table state
      columnWidths: {} // width of each column that has been modified
    };
  },
  mounted: function () {
    this.getTableState(); // IMPORTANT! this loads the data for the table
    this.getColumnWidths();
  },
  methods: {
    sort: function (sort) {
      // if the sort field is the same, toggle it, otherwise set it to default (true)
      this.tableDesc = this.tableSortField === sort ? !this.tableDesc : true;
      this.tableSortField = sort;
      this.loadData(this.tableSortField, this.tableDesc);
      this.saveTableState();
    },
    initializeColDragDrop: function () {
      setTimeout(() => { // wait for columns to render
        draggableColumns = Sortable.create(this.$refs.draggableColumns, {
          animation: 100,
          filter: '.ignore-element',
          preventOnFilter: false, // allow clicks within the ignored element
          onMove: (event) => { // col header is being dragged
            // don't allow a column to be dropped in the far left column
            return !event.related.classList.contains('ignore-element');
          },
          onEnd: (event) => { // dragged col header was dropped
            // nothing has changed, so don't do stuff
            if (event.oldIndex === event.newIndex) { return; }

            // update the headers to the new order
            let oldIdx = event.oldIndex;
            let newIdx = event.newIndex;

            // account for the index of the action column
            if (this.actionColumn) {
              oldIdx--;
              newIdx--;
            }

            let element = this.computedColumns[oldIdx];
            this.computedColumns.splice(oldIdx, 1);
            this.computedColumns.splice(newIdx, 0, element);

            this.saveTableState();
          }
        });
      });
    },
    initializeColResizable: function () {
      if (tableDestroyed) {
        $(`#${this.id}`).colResizable({ disable: true });
        tableDestroyed = false;
      }

      setTimeout(() => { // wait for columns to render
        let options = {
          minWidth: 50,
          headerOnly: true,
          resizeMode: 'overflow',
          hoverCursor: 'col-resize',
          removePadding: false,
          onResize: (event, col, colIdx) => {
            // account for the index of the action column
            if (this.actionColumn) { colIdx--; }

            let column = this.computedColumns[colIdx];

            if (column) {
              column.width = col.w;
              this.columnWidths[column.id] = col.w;
              this.saveColumnWidths();
            }

            // recalculate table width
            let tableWidth = 0;
            for (let column of this.computedColumns) {
              tableWidth += column.width;
            }
            this.tableWidth = tableWidth;
          }
        };

        // don't allow the action column to be resized
        if (this.actionColumn) {
          options.disabledColumns = [0];
        }

        $(`#${this.id}`).colResizable(options);
      });
    },
    getTableState: function () {
      UserService.getState(this.tableStateName)
        .then((response) => {
          if (response.data && response.data.order && response.data.visibleHeaders) {
            // there is a saved table state for this table
            // so apply it to sortField, desc, and column order
            this.tableSortField = response.data.order[0][0];
            this.tableDesc = response.data.order[0][1] === 'desc';
            for (let c of response.data.visibleHeaders) {
              for (let column of this.columns) {
                if (column.id === c) {
                  this.computedColumns.push(column);
                }
              }
            }
          } else {
            this.computedColumns = this.columns;
          }

          this.loadData(this.tableSortField, this.tableDesc);
          this.initializeColDragDrop();
        })
        .catch(() => {
          // if there's an error getting the table state,
          // just use the passed in columns
          this.computedColumns = this.columns;
          this.initializeColDragDrop();
        });
    },
    saveTableState: function () {
      let tableState = {
        order: [[this.tableSortField, this.tableDesc === true ? 'desc' : 'asc']],
        visibleHeaders: []
      };

      for (let column of this.computedColumns) {
        tableState.visibleHeaders.push(column.id);
      }

      UserService.saveState(tableState, this.tableStateName);
    },
    getColumnWidths: function () {
      UserService.getState(this.tableWidthsStateName)
        .then((response) => {
          this.columnWidths = response.data || {};
          let tableWidth = 0;
          for (let column of this.computedColumns) {
            for (let c in this.columnWidths) {
              if (column.id === c) {
                column.width = this.columnWidths[c];
              }
            }
            tableWidth += column.width;
          }
          this.tableWidth = tableWidth;
          this.initializeColResizable();
        })
        .catch(() => {
          // don't do anything, just use the supplied widths
          this.initializeColResizable();
        });
    },
    saveColumnWidths: function () {
      UserService.saveState(this.columnWidths, this.tableWidthsStateName);
    }
  },
  beforeDestroy: function () {
    tableDestroyed = true;
    if (draggableColumns) { draggableColumns.destroy(); }
  }
};
</script>
