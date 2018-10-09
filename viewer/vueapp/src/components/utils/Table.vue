<template>

  <!-- TODO persist column order and widths -->
  <table v-if="columns && columns.length"
    class="table table-striped"
    :class="tableClasses"
    :id="id">
    <thead>
      <tr ref="draggableColumns">
        <!-- TODO col config buttons -->
        <th v-if="actionColumn"
          class="ignore-element">
          &nbsp;
        </th>
        <th v-for="column in columns"
          :key="column.name"
          v-b-tooltip.hover
          :title="column.help"
          @click="sortEvent(column.sort)"
          :style="{'width': column.width + 'px'}"
          :class="{'cursor-pointer':column.sort}">
          {{ column.name }}
          <span v-if="column.sort">
            <span v-show="sortField === column.sort && !desc" class="fa fa-sort-asc"></span>
            <span v-show="sortField === column.sort && desc" class="fa fa-sort-desc"></span>
            <span v-show="sortField !== column.sort" class="fa fa-sort"></span>
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
        <td v-for="(column, index) in columns"
          :key="column.dataField+index">
          <span v-if="!column.dataFunction">
            {{ item[column.dataField] }}
          </span>
          <span v-else>
            {{ column.dataFunction(item[column.dataField]) }}
          </span>
        </td>
      </tr>
      <tr v-if="noResults && !data.length">
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
import Sortable from 'sortablejs';
import '../../../../public/colResizable.js';

let draggableColumns;

export default {
  name: 'MolochTable',
  props: {
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
      type: Array,
      required: true
    },
    noResults: { // whether or not to dispaly a no results row if data array is empty
      type: Boolean,
      default: false
    },
    sortEvent: { // event to fire when a column is clicked for sorting
      type: Function,
      required: true
    },
    sortField: { // the field that the query is sorting by
      type: String,
      required: true
    },
    desc: { // the direction the query is sorting by
      type: Boolean,
      required: true
    }
  },
  mounted: function () {
    this.initializeColDragDrop();
    this.initializeColResizable();
    // TODO get saved widths for table columns and set it to override widths supplied by parent
  },
  methods: {
    initializeColDragDrop: function () {
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

          // TODO loading?
          // update the headers to the new order
          let oldIdx = event.oldIndex;
          let newIdx = event.newIndex;

          // account for the index of the action column
          if (this.actionColumn) {
            oldIdx--;
            newIdx--;
          }

          let element = this.columns[oldIdx];
          this.columns.splice(oldIdx, 1);
          this.columns.splice(newIdx, 0, element);

          this.saveTableState();
        }
      });
    },
    initializeColResizable: function () {
      // TODO test table being destroyed
      let options = {
        minWidth: 50,
        headerOnly: true,
        resizeMode: 'overflow',
        hoverCursor: 'col-resize',
        onResize: (event, col, colIdx) => {
          // TODO loading?
          let column = this.columns[colIdx - 1]; // TODO need the -1?
          if (column) {
            column.width = col.w;
            this.colWidths[column.dbField] = col.w;

            this.saveColumnWidths();
          }
        }
      };

      // don't allow the action column to be resized
      if (this.actionColumn) {
        options.disabledColumns = [0];
      }

      $(`#${this.id}`).colResizable(options);
    },
    // TODO
    saveTableState: function () {
      console.log('do this');
    },
    // TODO
    saveColumnWidths: function () {
      console.log('do this');
    }
  },
  beforeDestroy: function () {
    if (draggableColumns) { draggableColumns.destroy(); }
  }
};
</script>
