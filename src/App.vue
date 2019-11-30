<template>
    <div class="container">
        <div>
            <v-chart :options="graph" :init-options="initOptions"/>
        </div>
        <button type="button" class="btn btn-light" @click="addNode">Add Node</button>
        <button type="button" class="btn btn-danger" @click="cleanGraphData">Reset</button>
        <button type="button" class="btn btn-warning" @click="deleteNode">Delete Node</button>
        <button type="button" class="btn btn-light" @click="exportData">Export Log Data</button>
        <div class="input-group input-group-sm mb-3">
            <input type="text" v-model="loopNumber" class="form-control" placeholder="Node Number" aria-label="Small" aria-describedby="inputGroup-sizing-sm">
            <button type="button" class="btn btn-light" @click="generateGraphWithNumber">Generate Graph</button>
        </div>
        <div class="row">
            <div v-for="item in graphData.nodes" v-bind:key="item.name" class="col">
                <a>Node {{item.name}}</a>
                <div class="w-100"></div>
                <table>
                <thead>
                    <tr>
                        <th>DST</th>
                        <th>DISTANCE</th>
                        <th>NEXT</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(k,v) in parents[item.name]" v-if="k!=null">
                        <td>{{v}}</td>
                        <td>{{distances[item.name][v]}}</td>
                        <td>{{k}}</td>
                    </tr>
                </tbody>
            </table>
            </div>
        </div>
        
    </div>
</template>

<script>
import ECharts from 'vue-echarts'
import { saveAs } from 'file-saver';

export default {
  components: {
    'v-chart': ECharts
  },
  methods:{
    initGraph(){
        this.graph = {title: {
                text: 'Disntance Vector Switch',
                subtext: 'Default layout',
                top: 'top',
                left: 'left'
            },
            tooltip: {},
            legend: [{
                
            }],
            animationDuration: 1500,
            animationEasingUpdate: 'quinticInOut',
            series : [
                {
                    name: 'Switch',
                    type: 'graph',
                    layout: 'none',
                    data: this.graphData.nodes,
                    links: this.graphData.links,
                    roam: true,
                    focusNodeAdjacency: true,
                    itemStyle: {
                        normal: {
                            borderColor: '#fff',
                            borderWidth: 1,
                            shadowBlur: 10,
                            shadowColor: 'rgba(0, 0, 0, 0.3)'
                        }
                    },
                    label: {
                        position: 'right',
                        formatter: 'switch-{b}'
                    },
                    lineStyle: {
                        color: 'source',
                        curveness: 0.3
                    },
                    emphasis: {
                        lineStyle: {
                            width: 10
                        }
                    }
                }
            ]
        }
    },
    initGraphData(){
        this.graphData = {
            nodes: [],
            links: []
        }
    },
    cleanGraphData(){
        this.graphData.nodes = [];
        this.graphData.links = [];
        this.initGraph();
        this.nodeid = 0;
    },
    addNode(){
        this.step ++;
        this.graphData.nodes.push({
            name: this.nodeid.toString(),
            x: Math.random(),
            y: Math.random(),
            value: this.nodeid
        });
        for(var i = this.nodeid; i> 0 ;i--){
            this.graphData.links.push({
                source: (i - 1).toString(),
                target: this.nodeid.toString(),
                value: Math.floor(Math.random() * 10) + 1,
                label: {
                    show: true,
                    formatter: '{@score}'
                }
            });
        }
        this.refreshRouteTable();
        this.nodeid += 1;
    },
    refreshRouteTable(){
        this.graphData.nodes.forEach(e => {
            var o = this.calcRouteTable(e);
            this.distances[e.name]= o.distances;
            this.parents[e.name] = o.parents;
            
        });

        this.historyData[this.step] = [];
        for(var item in this.graphData.nodes){
            this.historyData[this.step][item] = [];
            for(var k in this.parents[item]){
                var v = this.parents[item][k];
                if(v!=null){
                    this.historyData[this.step][item].push({
                        dst: k,
                        distance: this.distances[item][k],
                        next: v
                    });
                }
            }
        }
    },
    calcRouteTable(source){
        var vertexes = this.graphData.nodes;
        var edges = [];
        this.graphData.links.forEach(e => {
            edges.push({
                source: e.source,
                target: e.target,
                value: e.value,
            });
            edges.push({
                source: e.target,
                target: e.source,
                value: e.value,
            });
        });
        var distances = {};
        var parents = {}
        var c;
        for(var i = 0; i<vertexes.length ;i++){
            distances[vertexes[i].name] = Infinity;
            parents[vertexes[i].name] = null;
        }
        distances[source.name] = 0;
        for(var i =0 ; i< vertexes.length - 1 ; i++) {
            for (var j = 0; j < edges.length; j++) {
                    c = edges[j];
                    if (distances[c.source] + c.value < distances[c.target]) {
                            distances[c.target] = distances[c.source] + c.value;
                            parents[c.target] = c.source;
                    }
            }
        }
        for(var k = 0; k<edges.length ; k++) {
            if(parents[k] == source.name){
                parents[k] = k;
            }
        }
        return {parents, distances};
    },
    deleteNode(){
        if(this.graphData.nodes.length == 0){
            return;
        }
        var nodeid = this.nodeid-1;
        for(var i = 0;i<this.graphData.links.length; i ++){
            if(this.graphData.links[i].source == nodeid.toString()){
                this.graphData.links.pop(i);
            }
            if(this.graphData.links[i].target == nodeid.toString()){
                this.graphData.links.pop(i);
            }
        }
        for(var i = 0;i<this.graphData.nodes.length; i ++){
            if(this.graphData.nodes[i].name == nodeid.toString()){
                this.graphData.nodes.pop(i);
            }
        }
        this.nodeid --;
        this.step ++;
        this.refreshRouteTable();
    },
    exportData(){
        var printstr = '';
        for(var step = 1; step < this.historyData.length ; step++){
            var nodeData = this.historyData[step];
            printstr += `STEP ${step} \n`
            for( var node = 0; node < nodeData.length ; node++){
                var lineData = nodeData[node];
                printstr += `Node ${node}\n`;
                printstr += `DST DISTANCE NEXT \n`;
                for( var line = 0; line< lineData.length ; line ++){
                    var item = lineData[line];
                    printstr += `${item.dst}    ${item.distance}        ${item.next} \n`
                };
            }
            printstr += `----------------------- \n`
            printstr += `\n`
        }
        var blob = new Blob([printstr], {type: "text/plain;charset=utf-8"});
        saveAs(blob, "historyData.txt");
    },
    generateGraphWithNumber(){
        this.cleanGraphData();
        for(var i = 0; i< this.loopNumber ;i++){
            this.addNode();
        }
    }
  },
  mounted(){
      this.initGraphData();
      this.initGraph();
  },
  data () {
    return {
      graph:{},
      graphData: {},
      nodeid: 0, 
      distances: {},
      parents: {},
      initOptions: {
        renderer: 'svg'
      },
      historyData: [],
      step: 0,
      loopNumber: 10
    }
  }
}
</script>

<style>

</style>