<!--
Copyright Yahoo Inc.
SPDX-License-Identifier: Apache-2.0
-->
<template>

  <div class="container-fluid mt-2">

    <arkime-loading v-if="initialLoading && !error">
    </arkime-loading>

    <arkime-error v-if="error"
      :message="error">
    </arkime-error>

    <div v-show="!error">

      <arkime-paging v-if="stats"
        class="mt-1 ml-2"
        :info-only="true"
        :records-total="recordsTotal"
        :records-filtered="recordsFiltered">
      </arkime-paging>

      <arkime-table
        id="esRecoveryTable"
        :data="stats"
        :loadData="loadData"
        :columns="columns"
        :no-results="true"
        :action-column="true"
        :desc="query.desc"
        :sortField="query.sortField"
        :no-results-msg="`No results match your search.${cluster ? 'Try selecting a different cluster.' : ''}`"
        page="esRecovery"
        table-state-name="esRecoveryCols"
        table-widths-state-name="esRecoveryColWidths"
        table-classes="table-sm table-hover text-right small mt-2">
      </arkime-table>

    </div>

  </div>

</template>

<script>
import Utils from '../utils/utils';
import ArkimeTable from '../utils/Table';
import ArkimeError from '../utils/Error';
import ArkimeLoading from '../utils/Loading';
import ArkimePaging from '../utils/Pagination';

let reqPromise; // promise returned from setInterval for recurring requests
let respondedAt; // the time that the last data load successfully responded

export default {
  name: 'EsRecovery',
  props: [
    'user',
    'dataInterval',
    'recoveryShow',
    'refreshData',
    'searchTerm',
    'cluster'
  ],
  components: {
    ArkimeTable,
    ArkimeError,
    ArkimePaging,
    ArkimeLoading
  },
  data: function () {
    return {
      stats: null,
      error: '',
      initialLoading: true,
      totalValues: null,
      averageValues: null,
      recordsTotal: null,
      recordsFiltered: null,
      query: {
        filter: this.searchTerm || undefined,
        sortField: 'index',
        desc: false,
        show: this.recoveryShow || 'notdone',
        cluster: this.cluster || undefined
      },
      columns: [ // es indices table columns
        // default columns
        { id: 'index', name: 'Index', classes: 'text-left', sort: 'index', default: true, width: 200 },
        { id: 'shard', name: 'Shard', sort: 'shard', default: true, width: 80 },
        { id: 'time', name: 'Time', sort: 'time', default: true, width: 80 },
        { id: 'type', name: 'Type', sort: 'type', default: true, width: 100 },
        { id: 'stage', name: 'Stage', sort: 'stage', default: true, width: 100 },
        { id: 'source_host', name: 'Src Host', classes: 'text-left', sort: 'source_host', default: false, width: 200 },
        { id: 'source_node', name: 'Src Node', classes: 'text-left', sort: 'source_node', default: true, width: 120 },
        { id: 'target_host', name: 'Dst Host', classes: 'text-left', sort: 'target_host', default: false, width: 200 },
        { id: 'target_node', name: 'Dst Node', classes: 'text-left', sort: 'target_node', default: true, width: 120 },
        { id: 'files', name: 'Files', sort: 'files', default: false, width: 100 },
        { id: 'files_recovered', name: 'Files Recovered', sort: 'files_recovered', default: false, width: 100 },
        { id: 'files_percent', name: 'Files %', sort: 'files_percent', default: true, width: 80 },
        { id: 'files_total', name: 'Files total', sort: 'files_total', default: false, width: 100 },
        { id: 'bytes', name: 'Bytes', sort: 'bytes', default: false, width: 100 },
        { id: 'bytes_recovered', name: 'Bytes Recovered', sort: 'bytes_recovered', default: false, width: 100 },
        { id: 'bytes_percent', name: 'Bytes %', sort: 'bytes_percent', default: true, width: 80 },
        { id: 'bytes_total', name: 'Bytes total', sort: 'bytes_total', default: false, width: 100 },
        { id: 'translog_ops', name: 'Translog', sort: 'translog_ops', default: false, width: 100 },
        { id: 'translog_ops_recovered', name: 'Translog Recovered', sort: 'translog_ops_recovered', default: false, width: 100 },
        { id: 'translog_ops_percent', name: 'Translog %', sort: 'translog_ops_percent', default: true, width: 100 }
      ]
    };
  },
  computed: {
    loading: {
      get: function () {
        return this.$store.state.loadingData;
      },
      set: function (newValue) {
        this.$store.commit('setLoadingData', newValue);
      }
    }
  },
  watch: {
    dataInterval: function () {
      if (reqPromise) { // cancel the interval and reset it if necessary
        clearInterval(reqPromise);
        reqPromise = null;

        if (this.dataInterval === '0') { return; }

        this.setRequestInterval();
      } else if (this.dataInterval !== '0') {
        this.loadData();
        this.setRequestInterval();
      }
    },
    recoveryShow: function () {
      this.query.show = this.recoveryShow;
      this.loadData();
    },
    refreshData: function () {
      if (this.refreshData) {
        this.loadData();
      }
    },
    cluster: function () {
      this.query.cluster = this.cluster;
      this.loadData();
    }
  },
  created: function () {
    // set a recurring server req if necessary
    if (this.dataInterval !== '0') {
      this.setRequestInterval();
    }
  },
  methods: {
    /* helper functions ------------------------------------------ */
    setRequestInterval: function () {
      reqPromise = setInterval(() => {
        if (respondedAt && Date.now() - respondedAt >= parseInt(this.dataInterval)) {
          this.loadData();
        }
      }, 500);
    },
    loadData: function (sortField, desc) {
      if (!Utils.checkClusterSelection(this.query.cluster, this.$store.state.esCluster.availableCluster.active, this).valid) {
        return;
      }

      this.loading = true;
      respondedAt = undefined;

      // this.query.all = 'true';
      this.query.filter = this.searchTerm;

      if (desc !== undefined) { this.query.desc = desc; }
      if (sortField) { this.query.sortField = sortField; }

      this.$http.get('api/esrecovery', { params: this.query })
        .then((response) => {
          respondedAt = Date.now();
          this.error = '';
          this.loading = false;
          this.initialLoading = false;
          this.stats = response.data.data;
          this.recordsTotal = response.data.recordsTotal;
          this.recordsFiltered = response.data.recordsFiltered;
        }, (error) => {
          respondedAt = undefined;
          this.loading = false;
          this.initialLoading = false;
          this.error = error.text || error;
        });
    }
  },
  beforeDestroy: function () {
    if (reqPromise) {
      clearInterval(reqPromise);
      reqPromise = null;
    }
  }
};
</script>
