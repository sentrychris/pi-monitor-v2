<template>
  <div>
    <PageTitle title="Dashboard"></PageTitle>
    <Loading :show="!loaded" :status="status" :message="message"></Loading>
    <transition name="fade">
      <div class="pb-2" v-if="loaded">
        <MainHeader title="Dashboard Overview" class="mt-5 mt-md-4"></MainHeader>
        <section id="overview" class="my-4">
          <div class="row">
            <div class="col-12 col-md-6 col-lg-3 mb-4 mb-lg-0">
              <div class="platform-header h-100">
                <StatCard title="Platform" color="dark" icon="server">
                  <template v-slot:content>
                    <strong>OS</strong> {{ platform.distro }}<br>
                    <strong>Kernel</strong> {{ platform.kernel }}<br>
                    <strong>Up</strong> {{ platform.uptime }}<br>
                  </template>
                </StatCard>
              </div>
            </div>
            <div class="col-12 col-md-6 col-lg-3 mb-4 mb-lg-0">
              <div class="cpu-header h-100">
                <StatCard  title="CPU" color="success" icon="tachometer-alt">
                  <template v-slot:content>
                    <strong>Temp</strong> {{ cpu.temp }}°C<br>
                    <strong>Usage</strong> {{ cpu.usage }}%<br>
                    <strong>Frequency</strong> {{ cpu.freq }}MHz<br>
                  </template>
                </StatCard>
              </div>
            </div>
            <div class="col-12 col-md-6 col-lg-3 mb-4 mb-md-0">
              <div class="memory-header h-100">
                <StatCard title="Memory" color="success" icon="server">
                  <template v-slot:content>
                    <strong>Used</strong> {{ mem.used }} GB<br>
                    <strong>Free</strong> {{ mem.free }} GB<br>
                    <strong>Total</strong> {{ mem.total }} GB<br>
                  </template>
                </StatCard>
              </div>
            </div>
            <div class="col-12 col-md-6 col-lg-3">
              <div class="disk-header h-100">
                <StatCard title="Disk" color="success" icon="hdd">
                  <template v-slot:content>
                    <strong>Used</strong> {{ disk.used }} GB<br>
                    <strong>Free</strong> {{ disk.free }} GB<br>
                    <strong>Total</strong> {{ disk.total }} GB<br>
                  </template>
                </StatCard>
              </div>
            </div>
          </div>
        </section>

        <section id="charts" class="bg-white p-4 my-4 shadow border-radius-5">
          <div class="card border-0">
            <div class="card-header bg-white text-muted">
              <fa-icon :icon="['fas', 'tasks']" class="mr-3"></fa-icon>Statistics
            </div>
            <div class="card-body">
              <div class="row">
                <div class="col-12 col-md-6 col-lg-3 ml-auto">
                  <Gauge title="Temperature" id="temp" :metric="cpu.temp" format="{y}°C"></Gauge>
                </div>
                <div class="col-12 col-md-6 col-lg-3 ml-auto">
                  <Gauge title="CPU" id="cpu" :metric="cpu.usage" format="{y}%"></Gauge>
                </div>
                <div class="col-12 col-md-6 col-lg-3 ml-auto">
                  <Gauge title="Memory" id="mem" :metric="mem_percent" format="{y}%"></Gauge>
                </div>
                <div class="col-12 col-md-6 col-lg-3 ml-auto">
                  <Gauge title="Disk" id="disk" :metric="disk_percent" format="{y}%"></Gauge>
                </div>
              </div>
            </div>
          </div>
        </section>

        <section id="metrics" class="bg-white p-4 my-4 shadow border-radius-5">
          <div class="row">
            <div class="col-12">
              <div class="card border-0">
                <div class="card-header bg-white text-muted">
                  <fa-icon :icon="['fas', 'cogs']" class="mr-3"></fa-icon>Top Processes
                </div>
                <div class="card-body">
                  <Graph id="processes" type="bar" title="Top Processes" :data="processesGraphData"></Graph>
                  <div class="mt-4">
                    <DataTable type="horizontal" :data="processes"></DataTable>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </section>
      </div>
    </transition>
  </div>
</template>

<script>
import {liveCpu} from '@/services/live-data';

import PageTitle from '@/components/common/PageTitle';
import MainHeader from '@/components/common/MainHeader';
import StatCard from '@/components/common/StatCard';
import DataTable from '@/components/common/DataTable';
import Loading from '@/components/common/Loading';
import Gauge from '@/components/charts/Gauge';
import Graph from '@/components/charts/Graph';

export default {
  data() {
    return {
      platform: {
        distro: '',
        kernel: '',
        uptime: '',
      },
      cpu: {
        temp: 0,
        freq: 0,
        usage: 0
      },
      mem: {
        used: 0,
        free: 0,
        total: 0,
      },
      disk: {
        used: 0,
        free: 0,
        total: 0,
      },
      mem_percent: 0,
      disk_percent: 0,
      processes: {},
      processesGraphData: [],
      loaded: false,
      status: false,
      message: '',
    }
  },
  created() {
    this.message = 'Retrieving system information, please wait...';
    this.getSystem();
    this.$bus.$on('api-disconnect', () => {
      this.status = 'error';
      this.message = 'Unable to connect, please wait...'
    });
    this.$bus.$on('api-reconnect', () => {
      this.status = 'success'
      this.message = 'Connection successful, reloading view...'
      this.getSystem();
    });
  },
  methods: {
    getSystem() {
      this.$api.request('/system/').then(response => {
        ['platform', 'cpu', 'mem', 'disk'].forEach(key => {
          if (typeof this[key] === 'undefined' || typeof response.data[key] === 'undefined') {
            throw new Error('Undefined metric in API response.');
          }
          Object.keys(this[key]).forEach(result => {
            this[key][result] = response.data[key][result];
          });
        });
        this.mem_percent = response.data.mem.percent;
        this.disk_percent = response.data.disk.percent;
        this.processes = response.data.processes;
        this.formatProcessesDataForGraphs();
        this.updateDataRegularly(5000);
        this.loaded = true;
      }).catch(error => {
        this.setError(error.message);
      });
    },
    formatProcessesDataForGraphs() {
      let response = [];
      this.processes.forEach(proc => {
        response.push({
          name: proc.name,
          data: [proc.mem]
        });
      });
      this.processesGraphData = response;
    },
    updateDataRegularly(interval) {
      setInterval(() => {
        liveCpu.get('temp').then(temp => {
          this.cpu.temp = temp;
          this.$bus.$emit('update-gauge-temp', {value: temp, id: 'temp'})
        });
      }, interval);
    },
    setError(message) {
      this.loaded = false;
      this.status = 'error';
      this.message = message;
    }
  },
  components: {
    StatCard,
    Gauge,
    Graph,
    PageTitle,
    DataTable,
    Loading,
    MainHeader
  }
}
</script>