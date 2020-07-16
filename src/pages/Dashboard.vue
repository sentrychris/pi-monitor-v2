<template>
    <div>
        <Title :title="'Dashboard | RaspiMon - A Lightweight Monitor for the Raspberry Pi'"></Title>

        <Loading :show="!loaded" :status="status" :message="message"></Loading>

        <transition name="fade">
            <div class="container" v-if="loaded">

                <Header :component="'Dashboard'" :status="connectionStatus" :message="apiMessage"></Header>

                <section id="overview" class="my-4">
                    <div class="row">
                        <div class="col-12 col-md-12 col-lg-6 mb-4 mb-lg-0">
                            <div class="platform-header">
                                <div class="card bg-dark border-0 shadow">
                                    <div class="card-header border-0 bg-dark text-white d-flex justify-content-between align-items-center w-100">
                                        <strong>Platform</strong>
                                        <i class="fa fa-server card-icon"></i>
                                    </div>
                                    <div class="card-body border-radius-5 bg-dark text-white">
                                        <strong>Distro</strong> {{ platform.distro }}<br>
                                        <strong>Kernel</strong> {{ platform.kernel }}<br>
                                        <strong>Uptime</strong> {{ platform.uptime }}<br>
                                    </div>
                                </div>
                            </div>
                        </div>
                        <div class="col-12 col-md-6 col-lg-3 mb-4 mb-lg-0">
                            <div class="cpu-header">
                                <div class="card bg-brand border-0 shadow">
                                    <div class="card-header bg-brand border-0 text-white d-flex justify-content-between align-items-center w-100">
                                        <strong>CPU</strong>
                                        <i class="fa fa-tachometer-alt card-icon"></i>
                                    </div>
                                    <div class="card-body border-radius-5 bg-brand text-white">
                                        <strong>Temp</strong> {{ cpu.temp }}°C<br>
                                        <strong>Usage</strong> {{ cpu.usage }}%<br>
                                        <strong>Frequency</strong> {{ cpu.freq }}MHz<br>
                                    </div>
                                </div>
                            </div>
                        </div>
                        <div class="col-12 col-md-6 col-lg-3">
                            <div class="disk-header">
                                <div class="card bg-brand border-0 shadow">
                                    <div class="card-header bg-brand border-0 text-white d-flex justify-content-between align-items-center w-100">
                                        <strong>Disk</strong>
                                        <i class="fa fa-hdd card-icon btt-1"></i>
                                    </div>
                                    <div class="card-body border-radius-5 bg-brand text-white">
                                        <strong>Used</strong> {{ disk.used }}GB<br>
                                        <strong>Free</strong> {{ disk.free }}GB<br>
                                        <strong>Total</strong> {{ disk.total }}GB<br>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </section>

                <section id="charts" class="bg-white p-4 my-4 shadow border-radius-5">
                    <div class="card border-0">
                        <div class="card-header bg-white text-muted">
                            <strong>Statistics</strong>
                        </div>
                        <div class="card-body">
                            <div class="row">
                                <div class="col-12 col-md-6 col-lg-4 ml-auto">
                                    <Gauge :title="'Temperature'" :id="'temp'" :metric="cpu.temp"
                                           :format="'{y}°C'"></Gauge>
                                </div>
                                <div class="col-12 col-md-6 col-lg-4 ml-auto">
                                    <Gauge :title="'CPU'" :id="'cpu'" :metric="cpu.usage" :format="'{y}%'"></Gauge>
                                </div>
                                <div class="col-12 col-md-12 col-lg-4 ml-auto">
                                    <Gauge :title="'Disk'" :id="'disk'" :metric="disk_percent" :format="'{y}%'"></Gauge>
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
                                    <strong>Top Processes</strong>
                                </div>
                                <div class="card-body">
                                    <Graph :id="'processes'" :title="'Top Processes'"
                                           :categories="processesGraphCategories" :data="processesGraphData">
                                    </Graph>
                                    <div class="mt-4">
                                        <Table :type="'horizontal'" :data="processes"></Table>
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
    import Gauge from "../components/Gauge";
    import Graph from "../components/Graph";
    import Table from "../components/Table";
    import Title from "../components/Title";
    import Loading from "../components/Loading";
    import Header from "../components/Header";

    export default {
        name: 'Dashboard',
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

                disk: {
                    used: 0,
                    free: 0,
                    total: 0,
                },
                disk_percent: 0,

                fan: {
                    status: '',
                },

                processes: {},
                processesGraphData: [],
                processesGraphCategories: [],

                // UI props
                loaded: false,
                status: false,
                message: '',

                // API props
                url: 'http://rowles.ddns.net:8888/system/',
                interval: null,
                connectionAttempt: 0,
                connectionStatus: false,
                apiMessage: '',
            }
        },
        created() {
            this.message = 'Retrieving data from API, please wait...';
            this.getSystem();
        },
        mounted() {
            setInterval(() => {
                this.checkApiConnection();
            }, 120000)
        },
        methods: {
            getSystem() {
                ++this.connectionAttempt;

                fetch(this.url).then(response => {
                    if (response.ok) {
                        this.connectionStatus = true;
                        this.apiMessage = 'connected';
                        this.message = 'Data successfully received, initialising dashboard...';
                        return response.json();
                    } else {
                        throw new Error('ERROR: (' + response.status + ') could not fetch system statistics.');
                    }
                }).then(json => {
                    if (json.data) {
                        setTimeout(() => {
                            ['platform', 'cpu', 'disk'].forEach(key => {
                                if (typeof this[key] === 'undefined' || typeof json.data[key] === 'undefined') {
                                    throw new Error('Undefined metric in API response.');
                                }

                                Object.keys(this[key]).forEach(result => {
                                    this[key][result] = json.data[key][result];
                                });
                            });

                            this.disk_percent = json.data.disk.percent;
                            this.processes = json.data.processes;

                            this.formatProcessesDataForGraphs();
                            this.formatProcessesCategoriesForGraphs();

                            this.loaded = true;
                        }, 1000);
                    } else {
                        throw new Error('Oh shit.')
                    }
                }).catch(error => {
                    if (this.connectionAttempt < 3) {
                        this.getSystem();
                    } else {
                        this.connectionStatus = false;
                        this.loaded = false;
                        this.status = 'error';
                        this.message = error.message;
                    }
                });
            },
            checkApiConnection() {
                console.log('checking API connection...');
                fetch(this.url).then(response => {
                    if (!response.ok) {
                        throw new Error('Unable to connect.');
                    } else {
                        console.log('API connected...');
                        this.connectionStatus = true;
                    }
                }).catch(error => {
                    this.connectionStatus = false;
                    this.apiMessage = error.message
                    console.log(this.apiMessage);
                })
            },
            formatProcessesDataForGraphs() {
                let response = [];
                this.processes.forEach(proc => {
                    response.push({
                        name: proc.name,
                        data: [parseInt(proc.mem)]
                    })
                });
                this.processesGraphData = response;
            },
            formatProcessesCategoriesForGraphs() {
                let response = [];
                this.processesGraphData.forEach(proc => {
                    response.push(proc.name);
                });
                this.processesGraphCategories = response;
            }
        },
        components: {
            Gauge,
            Graph,
            Title,
            Table,
            Loading,
            Header
        }
    }
</script>

<style>
    #api-status {
        font-size: 1rem;
    }

    .header-decor {
        font-size: 1rem;
    }

    .bg-brand {
        background: #41B883;
    }

    .card-icon {
        font-size: 3rem;
        color: #f5f5f5;
    }

    .btt-1 {
        bottom: -11px;
    }
</style>