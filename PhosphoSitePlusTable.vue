<template>
  <div>
    <DxDataGrid class="table"
                :data-source="tableData"
                :show-borders="true"
                :column-auto-width="true"
                :allow-column-resizing="true"
    >
      <DxColumn caption="Gene Name(s)" data-type="string" :calculate-cell-value="str => str.geneName.replaceAll(',', ', ')"  />
      <DxColumn caption="Uniprot ID(s)" data-type="string" :calculate-cell-value="str => str.uniprotId.replaceAll(',', ', ')"/>
      <DxColumn caption="Position" data-type="string" data-field="position"/>
      <DxColumn caption="#LT References" data-type="string" data-field="lowThroughputRefs"/>
      <DxColumn caption="#HT References" data-type="string" data-field="highThroughputRefs"/>



    </DxDataGrid>
  </div>
</template>

<script>
import {DxDataGrid, DxColumn} from 'devextreme-vue/data-grid';


export default {
  name: "PhosphoSitePlusTable",
  components: {
    DxDataGrid,
    // eslint-disable-next-line vue/no-unused-components
    DxColumn
  },
  props: {
    tableData: {
      type: Array,
      default: () => undefined
    }
  }
}
</script>

<style scoped>

</style>