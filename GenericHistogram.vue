<template>
  <div class="datahistclass"></div>
</template>

<script>
const d3 = require("d3v4");

export default {
  props: {
    chartData: {
      type: Array,
      default: undefined,
    },
    title: {
      type: String,
      default: "",
    },
    xlabel: {
      type: String,
      default: "",
    },
    plotHistogram: {
      type: Boolean,
      default: true,
    },
    plotKDE: {
      type: Boolean,
      default: false,
    },
    markMedianBar: {
      type: Boolean,
      default: false,
    },
    minWidth: {
      type: Number,
      default: 200,
    },
    minHeight: {
      type: Number,
      default: 200,
    },
    selectedLines: {
      type: Array,
      default: null,
    },
    margin: {
      type: Object,
      default: function () {
        return { top: 20, right: 10, bottom: 50, left: 50 };
      },
    },
  },
  data: function () {
    return {
      minValue: 0,
      maxValue: 1,
    };
  },
  watch: {
    chartData: function () {
      this.init();
    },
    selectedLines: function (newSelectedLines) {
      this.drawLines(newSelectedLines);
    },
  },
  methods: {
    drawPlot: function (data) {
      var sortedData = [...data];
      sortedData.sort();
      //console.log(data)

      this.maxValue = Math.max.apply(Math, sortedData);
      this.minValue = Math.min.apply(Math, sortedData);

      // console.log(max)
      // console.log((Math.round(max*10)/10)+0.1)
      // console.log(min)

      var nclass = 20;

      // ref: https://bl.ocks.org/d3noob/96b74d0bd6d11427dd797892551a103c

      var width = this.minWidth - this.margin.left - this.margin.right;
      var height = this.minHeight - this.margin.top - this.margin.bottom;

      // ceil / floor to a multiple of 0.1
      var maxX = Math.ceil(this.maxValue / 0.1) * 0.1;
      var minX = Math.floor(this.minValue / 0.1) * 0.1;

      // set the ranges
      var x = d3.scaleLinear().domain([minX, maxX]).rangeRound([0, width]);
      var y = d3.scaleLinear().range([height, 0]);

      var thresholds = x.ticks(nclass - 1);

      // set the parameters for the histogram
      var histogram = d3
        .histogram()
        .value(function (d) {
          return d;
        })
        .domain(x.domain())
        .thresholds(thresholds);

      nclass = x.ticks(nclass - 1).length;

      // append the svg object to the body of the page
      // append a 'group' element to 'svg'
      // moves the 'group' element to the top left margin
      var svg = d3
        .select(this.$el)
        .append("svg")
        .attr("class", "HistogramContainer")
        .attr("width", width + this.margin.left + this.margin.right)
        .attr("height", height + this.margin.top + this.margin.bottom)
        .append("g")
        .attr(
          "transform",
          "translate(" + this.margin.left + "," + this.margin.top + ")"
        );

      // group the data for the bars
      var bins = histogram(sortedData);
      var maxFrequency = d3.max(bins, function (d) {
        return d.length;
      });

      // Scale the range of the data in the y domain
      y.domain([0, maxFrequency]);

      if (this.plotHistogram) {
        var i = 0;
        // append the bar rectangles to the svg element
        svg
          .selectAll("rect")
          .data(bins)
          .enter()
          .append("rect")
          .attr("class", "bar")
          .attr("x", 1)
          .attr("transform", function (d) {
            return "translate(" + x(d.x0) + "," + y(d.length) + ")";
          })
          .attr("width", function (d) {
            return x(d.x1) - x(d.x0) - 1;
          })
          .attr("height", function (d) {
            return height - y(d.length);
          })
          .attr("id", function () {
            return "bar" + i++;
          })
          .style("fill", "#CCCCCC"); // blue: #0073CF
      }

      if (this.plotKDE) {
        // Kernel density estimate graph
        var bandwidth = ((maxX - minX) / nclass) * 2;
        var density = this.kde(
          this.epanechnikov(),
          thresholds,
          sortedData,
          bandwidth
        );

        var line = d3
          .line()
          .curve(d3.curveBasis)
          .x(function (d) {
            return x(d[0]);
          })
          .y(function (d) {
            return y(d[1]);
          });

        svg
          .append("path")
          .datum(density)
          .attr("fill", "none")
          .attr("stroke", "#000")
          .attr("stroke-width", 1.5)
          .attr("stroke-linejoin", "round")
          .attr("d", line);
      }

      // add the x Axis
      svg
        .append("g")
        .attr("transform", "translate(0," + height + ")")
        .call(d3.axisBottom(x));

      svg
        .append("text")
        .attr(
          "transform",
          "translate(" +
            width / 2 +
            " ," +
            (height + this.margin.top + 20) +
            ")"
        )
        .style("text-anchor", "middle")
        .text(this.xlabel);

      // add the y Axis
      svg.append("g").call(d3.axisLeft(y));

      svg
        .append("text")
        .attr("transform", "rotate(-90)")
        .attr("y", 0 - this.margin.left)
        .attr("x", 0 - height / 2)
        .attr("dy", "1em")
        .style("text-anchor", "middle")
        .text("Frequency");

      svg.append("g").attr("class", "lines");

      // colour median bar red
      if (this.markMedianBar) {
        var median = this.getMedian(sortedData);
        // console.log(median)
        var medBarId = this.findMedBar(maxX, minX, nclass, median);
        //console.log(medBarId)

        svg.select("#bar" + medBarId).style("fill", "#C4071B");
      }
    },
    kde: function (kernel, thresholds, data, bandwidth) {
      var binWidth = thresholds[1] - thresholds[0];
      return thresholds.map((t) => [
        t,
        d3.sum(data, function (d) {
          return kernel((t - d) / bandwidth) / bandwidth;
        }) * binWidth,
      ]);
    },
    epanechnikov: function () {
      return function (x) {
        return Math.abs(x) <= 1 ? 0.75 * (1 - x * x) : 0;
      };
    },
    drawLines: function (oLines) {
      var width = this.minWidth - this.margin.left - this.margin.right;
      var height = this.minHeight - this.margin.top - this.margin.bottom;

      // ceil / floor to a multiple of 0.1
      var maxX = Math.ceil(this.maxValue / 0.1) * 0.1;
      var minX = Math.floor(this.minValue / 0.1) * 0.1;

      var x = d3.scaleLinear().domain([minX, maxX]).rangeRound([0, width]);

      var allLines = d3.select(this.$el).select(".lines");
      allLines.selectAll("line").remove();

      allLines.selectAll("path").remove();

      var lines = allLines.selectAll("line").data(oLines).enter();

      lines
        .append("line")
        .attr("x1", function (d) {
          return x(d.value);
        })
        .attr("y1", 0)
        .attr("x2", function (d) {
          return x(d.value);
        })
        .attr("y2", height)
        .style("stroke-width", 2)
        .style("stroke", function (d) {
          return d.color;
        })
        .style("stroke-dasharray", function (d) {
          return d.dash;
        })
        .style("fill", "none");

      lines
        .append("path")
        .attr("d", function (d) {
          return d.marker;
        })
        .attr("transform", function (d) {
          return "translate(" + x(d.value) + "," + height * 0.25 + ")";
        })
        .style("fill", function (d) {
          return d.color;
        });
    },
    getMedian: function (valarray) {
      var medval;
      if (valarray.length % 2 === 0) {
        // if even
        var val1where = valarray.length / 2;
        var val2where = valarray.length / 2 + 1;
        var val1 = valarray[val1where];
        var val2 = valarray[val2where];
        medval = (val1 + val2) / 2;
      } else {
        var where = (valarray.length + 1) / 2;
        medval = valarray[where];
      }
      return medval;
    },
    findMedBar: function (maxX, minX, nclass, median) {
      var size = (maxX - minX) / nclass;
      var found, i;
      for (i = 0; i <= nclass; i++) {
        if (minX + size * i < median && median < minX + size * (i + 1)) {
          found = i;
        }
      }
      return found;
    },
    init: function () {
      d3.select(this.$el).select("svg").remove();
      var oData = this.chartData;
      if (typeof oData !== "undefined" && oData.length > 0) {
        this.drawPlot(oData);
      }
    },
  },
  mounted: function () {
    this.init();
  },
};
</script>

<style>
@import "./GenericHistogram.css.prdb";
</style>
