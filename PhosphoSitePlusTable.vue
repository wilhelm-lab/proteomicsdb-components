<template>
  <div>
    <DxDataGrid class="table"
                :data-source="tableData"
                :show-borders="true"
                :column-auto-width="true"
                :allow-column-resizing="true"
    >
      <DxColumn caption="Gene Name(s)" data-type="string" :allow-sorting="true"
                :calculate-cell-value="d => d.geneName.replaceAll(',', ', ')"/>
      <DxColumn caption="Uniprot ID(s)" data-type="string" :allow-sorting="true"
                :calculate-cell-value="d => d.uniprotId.replaceAll(',', ', ')"/>
      <DxColumn caption="Position" data-type="string" :allow-sorting="true"
                :calculate-cell-value="d => {return {uniprotId: d.uniprotId, position: d.position}}"
                cell-template="pspTemplate"/>
      <DxColumn caption="#LT References" data-type="string" :allow-sorting="true" data-field="lowThroughputRefs"/>
      <DxColumn caption="#HT References" data-type="string" :allow-sorting="true" data-field="highThroughputRefs"/>

      <template #pspTemplate="{ data }">
        <a :href="'http://www.phosphosite.org/uniprotAccAction?id=' + data.value.uniprotId " target="_blank">
          {{ data.value.position }}</a>

      </template>

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