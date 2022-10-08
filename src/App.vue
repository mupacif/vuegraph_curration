<template>
  <div id="app">
    <ag-grid-vue
      class="ag-theme-alpine"
      style="height: 500px"
      :columnDefs="columnDefs.value"
      :rowData="rowData.value"
      :defaultColDef="defaultColDef"
      rowSelection="multiple"
      animateRows="true"
      @cell-clicked="cellWasClicked"
      @grid-ready="onGridReady"
    >
    </ag-grid-vue>
  </div>
</template>

<script>
import { AgGridVue } from 'ag-grid-vue3'; // the AG Grid Vue Component
import { reactive, onMounted, ref } from 'vue';
import { of, map, Observable, catchError } from 'rxjs';
import { ajax } from 'rxjs/ajax';
import 'ag-grid-community/styles/ag-grid.css'; // Core grid CSS, always needed
import 'ag-grid-community/styles/ag-theme-alpine.css'; // Optional theme CSS

const yourQuery = {
  query: `{
    subgraphs {
      id
      website
      displayName
      signalAmount
      currentSignalledTokens
      signalledTokens
      nameSignalCount
      nameSignalAmount
      codeRepository
      createdAt
      creatorAddress
    }
  }`,
};

export default {
  name: 'App',
  components: {
    AgGridVue,
  },
  setup() {
    const gridApi = ref(null); // Optional - for accessing Grid's API

    // Obtain API from grid's onGridReady event
    const onGridReady = (params) => {
      gridApi.value = params.api;
    };

    const rowData = reactive({}); // Set rowData to Array of Objects, one Object per Row

    // Each Column Definition results in one Column.
    const columnDefs = reactive({
      value: [
        { field: 'displayName', headerName: 'Nom' },
        { field: 'currentSignalledTokens', headerName: 'signals' },
      ],
    });

    // DefaultColDef sets props common to all Columns
    const defaultColDef = {
      sortable: true,
      filter: true,
      flex: 1,
    };

    // Example load data from sever
    onMounted(() => {
      fetch('https://www.ag-grid.com/example-assets/row-data.json')
        .then((result) => result.json())
        .then((remoteRowData) => (rowData.value = remoteRowData));

      const users = ajax({
        url: 'https://api.thegraph.com/subgraphs/name/graphprotocol/graph-network-mainnet',
        crossDomain: true,
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: {
          query: `{
            subgraphs(first: 50, orderDirection: desc, orderBy: currentSignalledTokens,skip:0,
            where: {active: true, displayName_not: "", entityVersion: 2}) {
              id
              website
              displayName
              signalAmount
              currentSignalledTokens
              signalledTokens
              nameSignalCount
              nameSignalAmount
              codeRepository
              createdAt
              creatorAddress
            }
          }`,
        },
      }).pipe(
        map(({ response }) => response.data),
        catchError((error) => {
          console.log('error: ', error);
          return of(error);
        })
      );
      users.subscribe({
        next: ({ subgraphs }) => (rowData.value = subgraphs),
        error: (err) => console.log(err),
      });
    });

    return {
      onGridReady,
      columnDefs,
      rowData,
      defaultColDef,
      cellWasClicked: ({ data }) => {
        // Example of consuming Grid Event
        const URL = `https://thegraph.com/explorer/subgraph?id=${data.id}`;
        window.open(URL, '_blank');
        console.log('cell was clicked', data);
      },
      deselectRows: () => {
        gridApi.value.deselectAll();
      },
    };
  },
};
</script>
<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
