<template>
  <div>
    <div id="expressionBarsAxisTop" class="expressionBarsAxisTop"></div>
    <v-card
      id="expressionBarsChart"
      class="expressionBarsChart"
      style="max-height: 650px; overflow-y: auto"
      flat
    >
    </v-card>
    <div id="expressionBarsAxisBottom" class="expressionBarsAxisBottom"></div>
  </div>
</template>

<script>
const d3 = require("d3");
export default {
  components: {},
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
    data: {
      type: Array,
      default: () => [],
    },
    barAttributes: {
      type: Object,
      default: () => {},
    },
    svgAttributes: {
      type: Object,
      default: () => {},
    },
  },
  data: function () {
    return {
      svg: null,
      barGroup: null,
    };
  },
  watch: {
    data: function (newData) {
      if (newData) {
        this.redraw();
      }
    },
  },
  methods: {
    redraw: function () {
      var oData = this.data;
      var oSvgAttributes = this.svgAttributes;
      var oBarAttributes = this.barAttributes;

      var width = oSvgAttributes.width;
      var margin = oSvgAttributes.margin;
      var cBlue = "#008FD3";

      d3.select(this.$el).selectAll("svg").remove();

      var svg = d3
        .select(this.$el)
        .select(".expressionBarsChart")
        .append("svg")
        .attr("class", "ExpressionBars")
        .attr("width", width + margin.left + margin.right)
        .attr(
          "height",
          oBarAttributes.height * oData.length +
            margin.chart.top +
            margin.bottom
        );
      var content = svg
        .append("g")
        .attr(
          "transform",
          "translate(" + margin.left + "," + margin.chart.top + ")"
        )
        .append("g")
        .attr("style", "overflow:auto;");

      this.svg = svg;

      var xMaxBasedOnMaxNormalizedIntensity = d3.max(
        oData,
        function getMaxNormIntensity(d) {
          return d.MAX_NORMALIZED_INTENSITY;
        }
      );
      var xMaxBasedOnNormalizedIntensity = d3.max(
        oData,
        function getMaxOfNormIntensity(d) {
          return oBarAttributes.width(d);
        }
      );
      var xMax = d3.max([
        xMaxBasedOnMaxNormalizedIntensity,
        xMaxBasedOnNormalizedIntensity,
      ]); // for ratios no MAX_NORMALIZED_INTENSITIES are given

      var xMinBasedOnMinNormalizedIntensity = d3.min(
        oData,
        function getMinNormIntensity(d) {
          return d.MIN_NORMALIZED_INTENSITY;
        }
      );
      var xMinBasedOnNormalizedIntensity = d3.min(
        oData,
        function getMinOfNormIntensity(d) {
          return oBarAttributes.width(d);
        }
      );
      var xMin = d3.min([
        xMinBasedOnNormalizedIntensity,
        xMinBasedOnMinNormalizedIntensity,
      ]); // for ratios no MAX_NORMALIZED_INTENSITIES are given

      var aYAxisLabels = oData.map(function getYAxisLabels(d) {
        return oBarAttributes.y(d);
      });

      var yScale = d3
        .scaleBand()
        .domain(d3.range(0, aYAxisLabels.length))
        //.domain(aYAxisLabels)
        .range([0, oBarAttributes.height * oData.length + 1]);

      var xScale;
      if (xMin > 0) {
        xScale = d3
          .scaleLinear()
          .domain([0, xMax])
          .range([0, oSvgAttributes.width]);
      } else {
        if (xMax > 0) {
          xScale = d3
            .scaleLinear()
            .domain([xMin, xMax])
            .range([0, oSvgAttributes.width]);
        } else {
          xScale = d3
            .scaleLinear()
            .domain([xMin, 0])
            .range([0, oSvgAttributes.width]);
        }
      }
      var yAxis = d3
        .axisLeft()
        .scale(yScale)
        .tickFormat(function getYAxisLabelsFromArray(d, i) {
          return aYAxisLabels[i];
        });

      var xAxisTop = d3.axisTop().scale(xScale);
      var xAxisBottom = d3.axisBottom().scale(xScale);

      var xAxisTopSvg = d3
        .select(this.$el)
        .select("#expressionBarsAxisTop")
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", margin.top + margin.chart.top + 5)
        .append("g");

      var xAxisDom = xAxisTopSvg.selectAll(".x.axis");
      if (xAxisDom.empty()) {
        xAxisDom = xAxisTopSvg.append("g").attr("class", "x axis");
      }
      xAxisDom
        .attr("transform", "translate(" + margin.left + "," + margin.top + ")")
        .call(xAxisTop);

      xAxisDom.selectAll(".tick text").text(function (d) {
        if (d < 0) return -d;
        else return d;
      });

      var xAxisBottomSvg = d3
        .select(this.$el)
        .select("#expressionBarsAxisBottom")
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", margin.top + margin.chart.top + 5)
        .append("g");

      var xAxisDomBottom = xAxisBottomSvg.selectAll(".x.axis");
      if (xAxisDomBottom.empty()) {
        xAxisDomBottom = xAxisBottomSvg.append("g").attr("class", "x axis");
      }
      xAxisDomBottom
        .attr("transform", "translate(" + margin.left + "," + 5 + ")")
        .call(xAxisBottom);

      xAxisDomBottom.selectAll(".tick text").text(function (d) {
        if (d < 0) return -d;
        else return d;
      });

      content
        .append("g")
        .classed("y_axis", true)
        .classed("axis", true)
        .call(yAxis)
        .append("text")
        .classed("label", true)
        .attr("x", -65)
        .attr("y", -10)
        .text(oSvgAttributes.y_axis_label)
        .attr("font-size", "13px");

      xAxisTopSvg
        .append("g")
        .classed("title", true)
        .append("text")
        .classed("label", true)
        .attr("x", margin.left)
        .attr("y", margin.top / 2)
        .text(oSvgAttributes.title)
        .attr("font-size", "14px")
        .attr("font-weight", "bold");

      // redraw
      var groups = content
        .selectAll("g.bar")
        .data(oData, function getBarAttributesId(d) {
          return oBarAttributes.id(d);
        });

      // add elements
      var g = groups
        .enter()
        .append("g")
        .classed("bar", true)
        .classed("clickEvent", this._bClickEvent);
      g.attr("transform", function translateBars(d, i) {
        return "translate(0," + (i * oBarAttributes.height + 2) + ")";
      });
      // reference group to handle mouse pointer in setBarClicked
      this.barGroup = g;

      g.append("rect");
      g.append("line").attr("class", "pointer");
      g.append("line").attr("class", "end_left");
      g.append("line").attr("class", "end_right");

      // styling included to make this self-contained, but should be outsourced to css
      // since data changes it must be reapplied
      svg
        .selectAll("g.axis path")
        .style("fill", "none")
        .style("stroke", "black")
        .style("shape-rendering", "crispEdges");
      svg
        .selectAll("g.axis line")
        .style("fill", "none")
        .style("stroke", "black");
      svg
        .selectAll("g.x_axis text")
        .style("font-family", "sans-serif")
        .style("font-size", function getFontSize() {
          return xMax > 1000 ? "9px" : "11px";
        });
      svg.selectAll(".label").style("font-size", "11px");
      svg
        .selectAll("g.y_axis text")
        .style("font-family", "sans-serif")
        .style("font-size", "11px");

      svg
        .selectAll("g.y_axis text.label")
        .style("font-family", "sans-serif")
        .style("font-size", "13px");

      // update all bars according to new scales
      svg
        .selectAll("g.bar rect")
        .classed("selected", false)
        .style("fill", cBlue)
        .style("stroke", "none")
        .style("stroke-width", 1)
        .attr("barId", function (d) {
          return d.SAP_SYNONYM;
        })
        .attr("x", function getValueForBar(d) {
          if (oBarAttributes.width(d) < 0) {
            return xScale(oBarAttributes.width(d)) + oBarAttributes.x;
          }
          if (oBarAttributes.x === xScale(0) + oBarAttributes.x) {
            // align bar to the left
            return oBarAttributes.x;
          }
          return xScale(0);
        })
        .attr("height", oBarAttributes.height - 5)
        .transition()
        .duration(1000)
        .attr("width", function getBarWitdh(d) {
          if (oBarAttributes.width(d) < 0) {
            return xScale(0) - xScale(oBarAttributes.width(d));
          }
          if (oBarAttributes.x === xScale(0) + oBarAttributes.x) {
            return xScale(oBarAttributes.width(d));
          }
          return xScale(oBarAttributes.width(d)) - xScale(0) - oBarAttributes.x;
        });

      g.append("text")
        .attr("y", oBarAttributes.height - 5)
        .attr("x", 360)
        .attr("font-size", "11px")
        .text(oBarAttributes.label);

      svg
        .selectAll("g.bar")
        .selectAll("line.pointer")
        .transition()
        .duration(1000)
        .attr("x1", function pointerGetX1(d) {
          if (d.SAMPLES > 1) {
            return xScale(d.MIN_NORMALIZED_INTENSITY);
          }
          return null;
        })
        .attr("y1", function pointerGetY1(d) {
          if (d.SAMPLES > 1) {
            return 6;
          }
          return null;
        }) // 6 is middle of 11 (height of bars)
        .attr("x2", function pointerGetX2(d) {
          if (d.SAMPLES > 1) {
            return xScale(d.MAX_NORMALIZED_INTENSITY);
          }
          return null;
        })
        .attr("y2", function pointerGetY2(d) {
          if (d.SAMPLES > 1) {
            return 6;
          }
          return null;
        })
        .style("stroke", "black")
        .style("stroke-width", "1px");

      svg
        .selectAll("g.bar")
        .selectAll("line.end_left")
        .transition()
        .duration(1000)
        .attr("x1", function endLeftGetX1(d) {
          if (d.SAMPLES > 1) {
            return xScale(d.MIN_NORMALIZED_INTENSITY);
          }
          return null;
        })
        .attr("y1", function endLeftGetY1(d) {
          if (d.SAMPLES > 1) {
            return 0;
          }
          return null;
        })
        .attr("x2", function endLeftGetX2(d) {
          if (d.SAMPLES > 1) {
            return xScale(d.MIN_NORMALIZED_INTENSITY);
          }
          return null;
        })
        .attr("y2", function endLeftGetY2(d) {
          if (d.SAMPLES > 1) {
            return 11;
          }
          return null;
        })
        .style("stroke", "black")
        .style("stroke-width", "1px");

      svg
        .selectAll("g.bar")
        .selectAll("line.end_right")
        .transition()
        .duration(1000)
        .attr("x1", function endRightGetX1(d) {
          if (d.SAMPLES > 1) {
            return xScale(d.MAX_NORMALIZED_INTENSITY);
          }
          return null;
        })
        .attr("y1", function endRightGetY1(d) {
          if (d.SAMPLES > 1) {
            return 0;
          }
          return null;
        })
        .attr("x2", function endRightGetX2(d) {
          if (d.SAMPLES > 1) {
            return xScale(d.MAX_NORMALIZED_INTENSITY);
          }
          return null;
        })
        .attr("y2", function endRightGetY2(d) {
          if (d.SAMPLES > 1) {
            return 11;
          }
          return null;
        })
        .style("stroke", "black")
        .style("stroke-width", "1px");

      g.on("click", this.barSelected);
      g.on("mouseover", function () {
        d3.select(this).style("cursor", "pointer");
      });

      groups.exit().remove();

      // update tipsy
      //if (oBarAttributes.tipsy) {
      //  $(sBarsId + ' .bar rect').tipsy({
      //    gravity: 'sw',
      //    html: true,
      //    title: function getBarTooltip() {
      //      var d = this.__data__;
      //      return oBarAttributes.tipsy(d);
      //    }
      //  });
      //}

      this.drawn = true;
    },
    resetBarSelection: function () {
      var cBlue = "#008FD3";
      d3.select(this.$el)
        .selectAll("g.bar rect")
        .classed("selected", false)
        .style("fill", () => cBlue)
        .style("stroke", () => "none");
    },
    toggleBar: function (element) {
      var bar = d3
        .select(this.$el)
        .selectAll("g.bar rect")
        .filter((d) => d.SAP_SYNONYM === element.SAP_SYNONYM);
      var cBlue = "#008FD3";
      var cBlueSelected = "#F0AB00";

      if (bar.classed("selected")) {
        bar
          .classed("selected", false)
          .style("fill", cBlue)
          .style("stroke", "none");
      } else {
        bar
          .classed("selected", true)
          .style("fill", cBlueSelected)
          .style("stroke", "black")
          .style("stroke-width", "1px");
      }
    },
    toggleBarSampleId: function (element) {
      var bar = d3
        .select(this.$el)
        .selectAll("g.bar rect")
        .filter((d) => d.SAMPLE_ID === element.SAMPLE_ID);
      this.toggleBarFinal(bar);
    },
    toggleBarTissueId: function (element) {
      var bar = d3
        .select(this.$el)
        .selectAll("g.bar rect")
        .filter((d) => d.TISSUE_ID === element.TISSUE_ID);
      this.toggleBarFinal(bar);
    },
    toggleBarFinal: function (bar) {
      var cBlue = "#008FD3";
      var cBlueSelected = "#F0AB00";

      if (bar.classed("selected")) {
        bar
          .classed("selected", false)
          .style("fill", cBlue)
          .style("stroke", "none");
      } else {
        bar
          .classed("selected", true)
          .style("fill", cBlueSelected)
          .style("stroke", "black")
          .style("stroke-width", "1px");
      }
    },
    barSelected: function barSelected(element) {
      if (element.SAMPLE_ID == -1) {
        this.$emit("barSelected", element);
        this.toggleBarTissueId(element);
      } else {
        this.$emit("sampleBarSelected", element);
        // this.toggleBarSampleId(element);
      }
    },
    getSVG: function () {
      var aPlots = [];
      aPlots.push(
        d3
          .select(this.$el)
          .select(".expressionBarsAxisTop")
          .selectAll("svg")
          .node()
      );
      aPlots.push(
        d3
          .select(this.$el)
          .select(".expressionBarsChart")
          .selectAll("svg")
          .node()
      );
      aPlots.push(
        d3
          .select(this.$el)
          .select(".expressionBarsAxisBottom")
          .selectAll("svg")
          .node()
      );
      return aPlots;
    },
  },
  mounted: function () {
    if (this.data !== undefined) {
      this.redraw();
    }
  },
};
</script>

<style scss>
#expressionBarsChart{
  max-height: 700px;
  overflow-y: auto;
}
.axis path,
.axis line {
  fill: none;
  stroke: #909090;
  shape-rendering: crispEdges;
}
<style>

