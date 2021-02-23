<template>
  <div
       class="fdrClass"
       ></div>
</template>

<script>
const d3 = require('d3v4')
export default {
  props: {
    width: {
      type: String,
      default: '100%'
    },
    height: {
      type: String,
      default: '100%;'
    },
    minWidthCSS: {
      type: [String, Number],
      default: '200px'
    },
    minHeightCSS: {
      type: [String, Number],
      default: '200px'
    },
    minWidth: {
      type: Number,
      default: 600
    },
    minHeight: {
      type: Number,
      default: 300
    },
    data: {
      type: Array,
      default: undefined
    },
    targetName: {
      type: String,
      default: null
    },
    targetPosition: {
      type: Number,
      default: undefined
    },
    decoyPosition: {
      type: Number,
      default: undefined
    },
    enrichment: {
      type: Number,
      default: undefined
    },
    qValue: {
      type: Number,
      default: undefined
    },
    xAxisLabel: {
      type: String,
      default: undefined
    },
    yAxisLabel: {
      type: String,
      default: undefined
    },
    logY: {
      type: Boolean,
      default: false
    },
    psmLevel: {
      type: Boolean,
      default: false
    }
  },
  data: function () {
    return {
    }
  },
  watch: {
    data: function (newData) {
      if (newData.length > 0) {
        this.drawPlot(newData)
      }
    },
    enrichment: function () {
      if(this.enrichment !== null && this.data !== null && this.data.length > 0)
        this.drawPlot(this.data)
    },
    targetName: function () {
      if(this.targetName !== null && this.data !== null && this.data.length > 0)
        this.drawPlot(this.data)
    }
  },
  methods: {
    getAttributeLineMapping: function getAttributeLineMapping() {
      return [{
        attribute: 'TARGET_COUNTS',
        name: 'target'
      }, {
        attribute: 'DECOY_COUNTS',
        name: 'decoy'
      }, {
        attribute: 'TP_COUNTS',
        name: 'target-decoy'
      }];
    },
    createLine: function createLine(x, y, oAttribute) {
      return d3.line()
      .curve(d3.curveLinear)
      .x(function xInterpolate(d) {
        return x(d.SCORE_BIN);
      })
      .y(function yInterpolates(d) {
        return y(d[oAttribute.attribute]);
      });
    },

    drawPlot: function (aData) {
      var oControl = this;
      var margin = {
        top: 40,
        right: 20,
        bottom: 50,
        left: 80
      };

      var width = this.minWidth - margin.left - margin.right
      var height = this.minHeight - margin.top - margin.bottom

      var sXAxisLabel = this.xAxisLabel;
      var sYAxisLabel = this.yAxisLabel;

      var aAttributeLineMapping = this.getAttributeLineMapping();

      var xMin = d3.min(aData, function max(d) {
        return d.SCORE_BIN;
      });

      var xMax = d3.max(aData, function max(d) {
        return d.SCORE_BIN;
      });

      var x = d3.scaleLinear()
      .domain([xMin, xMax])
      .range([0, width])
      .nice();

      var x2 = d3.scaleLinear()
      .domain([xMin, xMax])
      .range([0, width])
      .nice();

      var yMax = d3.max(aData, function max(d) {
        var aAttributes = [];
        // for every attribute in the mapping add it to the calculation of the yMax value
        for (var i = 0; i < aAttributeLineMapping.length; i++) {
          aAttributes[aAttributes.length] = d[aAttributeLineMapping[i].attribute];
        }
        return d3.max(aAttributes);
      });

      var y;
      if(this.logY) {
        y = d3.scaleLog()
        .domain([1, yMax])
        .range([height, 0])
        .nice();
      }else{
        y = d3.scaleLinear()
        .domain([0, yMax])
        .range([height, 0])
        .nice();
      }
      var color = d3.scaleOrdinal(d3.schemeCategory10);

      var xAxis = d3.axisBottom(x);

      var yAxis = this.logY ? d3.axisLeft(y) 
                          .ticks(8, function t(digit) {
                            return digit.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
                          })
                          : d3.axisLeft(y);

      var zoom = d3.zoom().scaleExtent([1,20]);

      var aLines = [];

      for (var i = 0; i < aAttributeLineMapping.length; i++) {
        aLines[aLines.length] = this.createLine(x, y, aAttributeLineMapping[i]);
      }

      d3.select(this.$el).selectAll('svg').remove()
      var svg = d3.select(this.$el).append('svg')
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)
      .attr('class', 'sapProteomicsdbFDRGraph');

      var container = svg.append('g')
      .style('pointer-events', 'none')
      .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');


      // clippath
      var sClipPathURL = 'url(#' + this.id + 'FDRClip)';
      container.append('clipPath')
      .attr('id', this.id + 'FDRClip')
      .append('rect')
      .attr('width', width)
      .attr('height', height);

      // zoom
      container.append('rect')
      .style('fill', 'none')
      .attr('width', width)
      .attr('height', height)
      .style('pointer-events', 'all')
      .style('cursor', '-webkit-grab')
      .call(zoom);

      // title
      container.append('text')
      .attr('class', 'title')
      .attr('text-anchor', 'middle')
      .attr('x', width / 2)
      .attr('y', -5)
      .text(this.targetName);

      // Q Value
      var sQValue = this.qValue ? 'q-value: ' + this.qValue.toExponential(1) : 'not identified';
      container.append('text')
      .attr('text-anchor', 'middle')
      .attr('x', width / 2)
      .attr('y', 15)
      .text(sQValue);

      // x Axis
      var x_axis = container.append('g')
      .attr('class', 'x axis')
      .attr('transform', 'translate(0,' + height + ')')
      .call(xAxis);

      container.append('text')
      .attr('class', 'x label')
      .attr('text-anchor', 'middle')
      .attr('x', width / 2)
      .attr('y', height + 30)
      .text(sXAxisLabel);

      // y Axis
      container.append('g')
      .attr('class', 'y axis')
      .call(yAxis);

      container.append('text')
      .attr('transform', 'rotate(-90)')
      .attr('y', -50)
      .attr('x', 0 - height / 2)
      .style('font-size', '12px')
      .style('text-anchor', 'middle')
      .text(sYAxisLabel);

      // lines
      container.selectAll('.lines')
      .data(aAttributeLineMapping)
      .enter()
      .append('path')
      .attr('clip-path', sClipPathURL)
      .style('stroke', function stroke(d, i) {
        return color(i);
      })
      .style('stroke-dasharray', function dasharray(d, i) {
        return i === 2 ? '2, 4' : '1, 1';
      })
      .attr('class', 'lines')
      .attr('d', function data(d, i) {
        return aLines[i](aData);
      });

      // vertical line decoy
      if (this.decoyPosition) {
        container.append('line')
        .attr('class', 'decoyLine')
        .attr('clip-path', sClipPathURL)
        .style('stroke', 'black')
        .style('stroke-dasharray', '2, 4')
        .attr('y1', y(0))
        .attr('y2', 25)
        .attr('x1', x(this.decoyPosition))
        .attr('x2', x(this.decoyPosition));

        container.append('circle')
        .attr('class', 'decoyCircle')
        .attr('clip-path', sClipPathURL)
        .attr('cx', x(this.decoyPosition))
        .attr('cy', y(0))
        .attr('r', 5)
        .style('fill', color(1));
      }

      // target marker
      if (this.targetPosition) {
        container.append('line')
        .attr('class', 'targetLine')
        .attr('clip-path', sClipPathURL)
        .style('stroke', 'black')
        .style('stroke-dasharray', '2, 4')
        .attr('y1', y(0))
        .attr('y2', 25)
        .attr('x1', x(this.targetPosition))
        .attr('x2', x(this.targetPosition));

        container.append('circle')
        .attr('class', 'targetCircle')
        .attr('clip-path', sClipPathURL)
        .attr('cx', x(this.targetPosition))
        .attr('cy', y(0))
        .attr('r', 5)
        .style('fill', color(0));
      }

      // Enrichment
      if (this.enrichment) {
        var nEnrichment = this.enrichment;
        container.append('text')
        .attr('class', 'enrichmentText')
        .attr('clip-path', sClipPathURL)
        .attr('x', x(nEnrichment > 1 ? this.targetPosition : this.decoyPosition) + 20)
        .attr('y', 40)
        .attr('text-anchor', 'middle')
        .attr('alignment-baseline', 'middle')
        .text(nEnrichment.toFixed(1) + 'x');

        container.append('line')
        .attr('class', 'enrichmentLine')
        .attr('clip-path', sClipPathURL)
        .style('stroke', 'black')
        .style('stroke-dasharray', '2, 4')
        .attr('y1', 40)
        .attr('y2', 40)
        .attr('x1', x(this.targetPosition))
        .attr('x2', x(this.decoyPosition));

        container.append('circle')
        .attr('class', 'enrichmentCircle')
        .attr('clip-path', sClipPathURL)
        .attr('cx', x(this.decoyPosition))
        .attr('cy', 40)
        .attr('r', 4)
        .style('fill', 'black');

        var iDirection = nEnrichment > 1 ? 1 : -1;
        var aTrianglePoints = [
        [x(this.targetPosition), 40],
        [x(this.targetPosition) - iDirection * 8, 37],
        [x(this.targetPosition) - iDirection * 8, 43]
        ];
        var sTrianglePoints = aTrianglePoints.join(',');
        container.append('polygon')
        .attr('class', 'enrichmentTriangle')
        .attr('clip-path', sClipPathURL)
        .attr('points', sTrianglePoints)
        .style('stroke', 'black')
        .style('fill', 'black');
      }

      // legend
      var nLegendLineWidth = 25;
      var nLegendRowHeight = 10;
      var nLegendOffset = 10;
      var existingLabels = this.data.map((x)=>{return(Object.keys(x))}).reduce((p,n) => {return(p.concat(n))}).reduce((p,n)=>{if(p.indexOf(n) != -1){return(p);}else{return(p.concat(n))}},[]);

      var aLegendLabels = ['target', 'decoy', 'target-decoy'];
      aLegendLabels = this.getAttributeLineMapping().filter((x)=> {return(existingLabels.indexOf(x.attribute) !== -1)}).map((x)=>{return(x.name)});

      var legend = container.selectAll('.legend')
      .data(aLegendLabels)
      .enter()
      .append('g')
      .attr('class', 'legend')
      .attr('transform', function transform(d, i) {
        var combined = nLegendRowHeight + nLegendOffset;
        var xPos = width - 125;
        var yPos = i * combined + nLegendOffset;
        return 'translate(' + xPos + ',' + yPos + ')';
      });

      legend.append('line')
      .attr('x1', 0)
      .attr('x2', nLegendLineWidth)
      .attr('y1', nLegendRowHeight / 2)
      .attr('y2', nLegendRowHeight / 2)
      .style('stroke', function stroke(d, i) {
        return color(i);
      })
      .style('stroke-dasharray', function dasharray(d, i) {
        if (i === 2) {
          return '2, 4';
        }
        return '1, 0';
      });

      legend.append('text')
      .attr('x', nLegendLineWidth + nLegendOffset)
      .attr('y', nLegendRowHeight - 1)
      .text(function text(d) {
        return d;
      });

      zoom.translateExtent([[0, 0], [width, height]])
          .extent([[0, 0], [width, height]])
          .on('zoom', function onZoom() {
        oControl.updateElements(container, x, aData, aLines, xAxis, zoom, width, x_axis,x2);
      });

      this._svg = svg;
    },
   // updateElements: function updateElements(svg, x, aData, aLines, xAxis, zoom, nWidth, x_axis) {
    updateElements: function updateElements(svg, x, aData, aLines, xAxis, zoom, nWidth, x_axis, x2) {
      // limit translate range
      var t = d3.event.transform;
      x.domain(t.rescaleX(x2).domain());

      x_axis
      .call(xAxis);

      svg.selectAll('path.lines').attr('d', function data(d, i) {
        return aLines[i](aData);
      });

      svg.select('.decoyLine')
      .attr('x1', x(this.decoyPosition))
      .attr('x2', x(this.decoyPosition));
      svg.select('.decoyCircle')
      .attr('cx', x(this.decoyPosition));

      svg.select('.targetLine')
      .attr('x1', x(this.targetPosition))
      .attr('x2', x(this.targetPosition));
      svg.select('.targetCircle')
      .attr('cx', x(this.targetPosition));

      var nEnrichment = this.enrichment;
      svg.select('.enrichmentText')
      .attr('x', x(nEnrichment > 1 ? this.targetPosition : this.decoyPosition) + 20);
      svg.select('.enrichmentLine')
      .attr('x1', x(this.targetPosition))
      .attr('x2', x(this.decoyPosition));
      svg.select('.enrichmentCircle')
      .attr('cx', x(this.decoyPosition));

      var iDirection = nEnrichment > 1 ? 1 : -1;
      var aTrianglePoints = [
      [x(this.targetPosition), 40],
      [x(this.targetPosition) - iDirection * 8, 37],
      [x(this.targetPosition) - iDirection * 8, 43]
      ];
      var sTrianglePoints = aTrianglePoints.join(',');
      svg.select('.enrichmentTriangle')
      .attr('points', sTrianglePoints);
    },

    getSVG: function getSVG() {
      return d3.select(this.$el).select('svg').node();
    }
  },
  mounted: function () {
    var oData = this.data
    if (oData !== [] && oData !== null && this.targetName != null)  {
      this.drawPlot(oData)
    }
  }
}
</script>

<style scss>
@import './FDRGraph.css.prdb'
</style>

