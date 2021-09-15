<template>
  <div id="volcanoplot" class="volcanoClass"></div>
</template>

<script>
const d3v4 = require("d3v4");
export default {
  props: {
    width: {
      type: String,
      default: "40%",
    },
    height: {
      type: String,
      default: "40%",
    },
    minWidthCSS: {
      type: [String, Number],
      default: "100",
    },
    minHeightCSS: {
      type: [String, Number],
      default: "100",
    },
    minWidth: {
      type: Number,
      default: 500,
    },
    minHeight: {
      type: Number,
      default: 500,
    },
    chartData: {
      type: Array,
      default: undefined,
    },
    busy: {
      type: Boolean,
      default: false,
    },
    xAxis: {
      type: String,
      default: "x-axis",
    },
    yAxis: {
      type: String,
      default: "y-axis",
    },
    mirroredSelection: {
      type: Boolean,
      default: true,
    },
    aSelectedGenesUp: {
      type: Object,
      default: undefined,
    },
    aSelectedGenesDown: {
      type: Object,
      default: undefined,
    },
  },
  data: function () {
    return {};
  },
  watch: {
    chartData: function (newData) {
      this.drawPlot(newData);
    },
  },
  methods: {
    drawPlot: function (data) {
      var that = this;

      var margin = {
        top: 10,
        right: 50,
        bottom: 50,
        left: 50,
      };

      var width = this.minWidth - margin.left - margin.right;
      var height = this.minHeight - margin.top - margin.bottom;

      d3v4.select(this.$el).selectAll("svg").remove();
      var svg = d3v4
        .select(this.$el)
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom);

      //  legend
      svg.append("g");

      svg
        .append("g")
        .attr(
          "transform",
          "translate(" + margin.left + ", " + margin.top + ")"
        );
      var g = svg
        .append("g")
        .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

      var oMinMaxPvalueFoldChange = this.minMaxXY(data);
      var iAbsoluteFoldChange = d3v4.max([
        Math.abs(oMinMaxPvalueFoldChange.fold_min),
        Math.abs(oMinMaxPvalueFoldChange.fold_max),
      ]);

      var xScale = d3v4
        .scaleLinear()
        .domain([-1 * iAbsoluteFoldChange, iAbsoluteFoldChange])
        .range([0, width]);

      var yScale = d3v4
        .scaleLinear()
        .domain([
          oMinMaxPvalueFoldChange.p_min * 0.95,
          oMinMaxPvalueFoldChange.p_max * 1.05,
        ])
        .range([height, 0]); // probably inverting!

      g.append("g")
        .attr("class", "axis axis--x")
        .attr("transform", "translate(0," + height + ")")
        .call(d3v4.axisBottom(xScale).ticks(5).tickFormat(d3v4.format(".1e")));

      g.append("text")
        .attr(
          "transform",
          "translate(" + width / 2 + " ," + (height + 35) + ")"
        )
        .style("text-anchor", "middle")
        .text(this.xAxis);

      g.append("g")
        .attr("class", "axis axis--y")
        .attr("transform", "translate(" + 0 + "," + 0 + ")")
        .call(d3v4.axisLeft(yScale).ticks(20));

      var fMidPValue =
        (oMinMaxPvalueFoldChange.p_max + oMinMaxPvalueFoldChange.p_min) / 2;
      g.append("text")
        .attr(
          "transform",
          "translate(" + -37 + " ," + yScale(fMidPValue) + ") rotate(-90)"
        )
        .style("text-anchor", "middle")
        .text(this.yAxis);

      var circle = g
        .append("g")
        .attr("class", "circle")
        .selectAll("circle")
        .data(data)
        .enter()
        .append("circle")
        .attr("transform", function t(d) {
          return "translate(" + xScale(d.x) + "," + yScale(d.y) + ")";
        })
        .attr("r", 3.5);

      function getSelectedKeys() {
        that.highlightCircle(
          d3v4
            .selectAll("circle")
            .data()
            .map((m) => {
              return m.GeneName;
            }),
          false
        );
        that.$emit("selection-wrapper", {
          aSelectedGeneNamesUp: d3v4
            .selectAll(".active")
            .data()
            .map(function m(d) {
              if (d.x >= 0) {
                return { genename: d.GeneName, y: d.y, x: d.x };
              }
              return null;
            })
            .filter(function f(d) {
              return d !== null;
            }),
          aSelectedGeneNamesDown: d3v4
            .selectAll(".active")
            .data()
            .map(function m(d) {
              if (d.x < 0) {
                return { genename: d.GeneName, y: d.y, x: d.x };
              }
              return null;
            })
            .filter(function f(d) {
              return d !== null;
            }),
        });
      }

      function brushmoved() {
        var brushSelection = d3v4.select(this);
        var s = d3v4.event.selection;
        if (brushSelection.attr("id") === "BrushR") {
          d3v4
            .select("#BrushL")
            .selectAll(".selection")
            .style("display", "none");
        } else {
          d3v4
            .select("#BrushR")
            .selectAll(".selection")
            .style("display", "none");
        }

        if (s === null) {
          // handle.attr('display', 'none')
          circle.classed("active", false);
        } else {
          var sxy = s.map(function m(edge) {
            return [xScale.invert(edge[0]), yScale.invert(edge[1])];
          });
          circle.classed("active", function c(d) {
            return (
              (sxy[0][0] <= d.x && // correct area of plot
                sxy[1][0] >= d.x &&
                sxy[0][1] >= d.y &&
                sxy[1][1] <= d.y) ||
              (that.mirroredSelection &&
                sxy[0][0] <= -d.x && // top left
                sxy[1][0] >= -d.x &&
                sxy[0][1] >= d.y &&
                sxy[1][1] <= d.y)
            );
          });
        }
      }

      var brushR = d3v4
        .brush()
        .extent([
          [xScale(0), 0],
          [width, height],
        ])
        .on("start brush", brushmoved)
        .on("end", getSelectedKeys);

      var brushL = d3v4
        .brush()
        .extent([
          [0, 0],
          [xScale(0), height],
        ])
        .on("start brush", brushmoved)
        .on("end", getSelectedKeys);

      g.append("g").attr("id", "BrushR").attr("class", "brush").call(brushR);

      g.append("g").attr("id", "BrushL").attr("class", "brush").call(brushL);

      function highlightCircle(aGeneNames, bFlag) {
        var t = d3v4.transition().duration(750).ease(d3v4.easeLinear);

        if (bFlag) {
          d3v4
            .selectAll("circle")
            .filter(function f(d) {
              return aGeneNames.includes(d.GeneName);
            })
            .transition(t)
            .attr("r", 10);
        } else {
          d3v4
            .selectAll("circle")
            .filter(function f(d) {
              return aGeneNames.includes(d.GeneName);
            })
            .transition(t)
            .attr("r", 3.5);
        }
      }

      function unSelectCircles(aGeneNames) {
        d3v4
          .selectAll(".active")
          .filter(function f(d) {
            return aGeneNames.includes(d.GeneName);
          })
          .classed("active", false);

        this.highlightCircle(aGeneNames, false);
      }

      that.highlightCircle = highlightCircle;
      that.unSelectCircles = unSelectCircles;
    },

    highlightCircle: function highlightCircle() {},

    unSelectCircles: function unSelectCircles() {},

    minMaxXY: function minMaxXY(data) {
      var out = {};
      out.p_min = d3v4.min(
        data.map(function m(d) {
          return d.y;
        })
      );
      out.p_max = d3v4.max(
        data.map(function m(d) {
          return d.y;
        })
      );
      out.fold_min = d3v4.min(
        data.map(function m(d) {
          return d.x;
        })
      );
      out.fold_max = d3v4.max(
        data.map(function m(d) {
          return d.x;
        })
      );

      return out;
    },
  },
  mounted: function () {
    d3v4.select(".sapProteomicsdbVolcanoPlot").select("svg").remove();

    var oData = this.chartData;

    if (typeof oData !== "undefined") {
      this.drawPlot(oData);
    }
  },
};
</script>

<style scss>
circle {
  fill-opacity: 0.2;
  transition: fill-opacity 250ms linear;
}
circle.active {
  stroke: #f00;
}
.volcanoClass {
}
</style>

