<template>
  <div class="lineplotClass"></div>
</template>

<script>
const d3 = require('d3')

export default {
  props: {
    minWidth: {
        type: Number,
        default: 200
    },
    minHeight: {
        type: Number,
        default: 200
    },
    curveParameters: {
        type: Array,
        default: undefined
    },
    dataPoints: {
        type: Array,
        default: undefined
    },
    properties: {
        type: Array,
        default: undefined
    },
    title: {
        type: String,
        default: undefined
    }
  },
  watch: {
    dataPoints: function () {
      this.init()
    }
  },
  
  methods: {
    getCurveFunction: function (oParameters) {
        var formula = oParameters.formula;
        for (var key in oParameters) {
            while (formula.search(key) > 0) {
                formula = formula.replace(key, oParameters[key].value);
            }
        }

        return Function('x', formula);
    },

    calculateCurvePoints: function (oParameters, aDataPoints) {
        var fCurve = this.getCurveFunction(oParameters);

        var min = d3.min(aDataPoints, function(d) {
            return d[0];
        });
        var max = Math.ceil(d3.max(aDataPoints, function(d) {
            return d[0];
        }));
        var aValues = [];

        var minLog = Math.log10(min);
        var maxLog = Math.log10(max);
        var step = 0.05;
        aValues.push(minLog);
        //        Math.floor((maxLog - minLog) / step);
        for (var j = minLog + step; j <= maxLog; j = j + step) {
            aValues.push(j);
        }
        aValues.push(maxLog);

        aValues = aValues.map(function(d) {
            return Math.pow(10, d);
        });
        var aData = aValues.map(function(d) {
            return [d, fCurve(d)];
        });
        return aData;
    },

    createCurveIdDictionary: function (aProperties) {
        var oDictionary = {}
        var treatmentDictionary = {}
        var numTreatments = 0
        for (var i = 0; i < aProperties.length; i++) {
            if (!(aProperties[i].TREATMENT in treatmentDictionary)) {
              treatmentDictionary[aProperties[i].TREATMENT] = ++numTreatments
            }
            oDictionary[aProperties[i].curveId] = treatmentDictionary[aProperties[i].TREATMENT];
        }
        return oDictionary;
    },

    highlightMouseOver: function (sId, x, y) {
        d3.select('#errorLine-' + sId).attr('class', 'highlightLines');
        d3.select('#legendLine-' + sId).attr('class', 'highlightLines');
        d3.select('#plotLine-' + sId).attr('class', 'highlightLines');
        d3.select('#legendText-' + sId).attr('font-weight', 'bold');
        d3.selectAll('#dots-' + sId).attr('transform', function(d) {
            return 'translate(' + x(d.x) + ',' + y(d.y) + ') scale(1)';
        });
        d3.selectAll('#dotsLegend-' + sId).attr('transform', 'translate(' + 25 / 2 + ',' + 10 / 2 + ') scale(1)'); // sry for hard coding
    },

    unHighlightMouseOver: function (sId, x, y) {
        d3.select('#errorLine-' + sId).attr('class', 'lines');
        d3.select('#legendLine-' + sId).attr('class', 'lines');
        d3.select('#plotLine-' + sId).attr('class', 'lines');
        d3.select('#legendText-' + sId).attr('font-weight', 'normal');
        d3.selectAll('#dots-' + sId).attr('transform', function(d) {
            return 'translate(' + x(d.x) + ',' + y(d.y) + ') scale(0.75)';
        });
        d3.selectAll('#dotsLegend-' + sId).attr('transform', 'translate(' + 25 / 2 + ',' + 10 / 2 + ') scale(0.75)');
    },

    drawPlot: function (aData, aPoints, aProperties, aParameters) {
        // assign each different treatment a number for graph coloring
        // aData is the curveData
        // aPoints are the plotted measurements
        var that = this;

        var oColorMapping = this.createCurveIdDictionary(aProperties);
        var margin = {
            top: 40,
            right: 250,
            bottom: 80,
            left: 50
        };

        var width = this.minWidth - margin.left - margin.right;
        var height = this.minHeight - margin.top - margin.bottom;
        
        // scale x Axis for highest within all series

        var x = d3.scaleLog().domain([d3.min(aData, function(d) {
            return d3.min(d, function(e) {
                return e[0];
            });
        }), d3.max(aData, function(d) {
            return d3.max(d, function(e) {
                return e[0];
            });
        })]).range([0, width]).nice();

        var yMaxPoints = 0;

        var yMaxCurve = 0;
        yMaxPoints = d3.max(aPoints, function(d) {
            return d3.max(d, function(e) {
                return e.y;
            });
        });
        yMaxCurve = d3.max(aData, function(d) {
            return d3.max(d, function(e) {
                return e[1];
            });
        });
        var y = d3.scaleLinear().domain([0, yMaxPoints > yMaxCurve ? yMaxPoints : yMaxCurve]).range([height, 0]).nice();

        var color = d3.scaleOrdinal(d3.schemeCategory10);

        var xAxis = d3.axisBottom(x).ticks(8, function ticks(digit) {
            return digit;
        });

        var yAxis = d3.axisLeft(y);

        var line = d3.line().curve(d3.curveLinear).x(function(d) {
            return x(d[0]);
        }).y(function(d) {
            return y(d[1]);
        });

        var svg = d3.select(this.$el).append('svg')
          .attr('class', 'LinePlot')
          .attr('width', width + margin.left + margin.right)
          .attr('height', height + margin.top + margin.bottom);

        this.svg = svg;

        svg = svg.append('g').attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');

        svg.append('g').attr('class', 'x axis').attr('transform', 'translate(0,' + height + ')').call(xAxis);

        svg.append('g').attr('class', 'y axis').call(yAxis);

        // Lines - color for different treatments, dash style for different biological replicates

        svg.append('g').attr('class', 'lines').selectAll('.line').data(aData).enter().append('path').filter(function(f) {
            return f.filter(function(g) {
                return g[1] > 0;
            });
        }).attr('id', function(d, i) {
            return 'plotLine-' + aProperties[i].curveId;
        }).style('stroke', function(d, i) {
            return color(oColorMapping[aProperties[i].curveId]);
        }).style('stroke-dasharray', function(d, i) {
            var x = aProperties[i].biologicalReplicate;
            return x * 3 + ',' + (x - 1) * 3;
        }).on('mouseover', function() {
            var sId = d3.select(this).attr('id').split('-')[1]; // next time: we do data binding correctly
            that.highlightMouseOver(sId, x, y);
        }).on('mouseout', function() {
            var sId = d3.select(this).attr('id').split('-')[1]; // next time: we do data binding correctly
            that.unHighlightMouseOver(sId, x, y);
        }).attr('d', line);
        var iPxOffSetErrorBar = 8;

        var errorGroup = svg.selectAll('g.error').attr('class', 'errorGroup').data(aParameters.map(function(x, i) {
            return {
                data: x['ED50, inflection'],
                formula: that.getCurveFunction(aParameters[i]),
                curveId: aProperties[i].curveId
            };
        }));

        // add elements
        var g = errorGroup.enter().append('g').filter(function(d) {
            return x(d.data.value) < width && x(d.data.value) > 0 && y(d.formula(d.data.value)) > 0 && y(d.formula(d.data.value)) < height;
        }).attr('class', 'errorBox').attr('id', function(d) {
            return 'errorLine-' + d.curveId;
        }).style('stroke', function(d) {
            return color(oColorMapping[d.curveId]);
        });

        g.append('line').attr('class', 'errorLine');
        g.append('line').attr('class', 'end_left').attr('x1', -1000).attr('x2', -1000).attr('y1', -1000).attr('y2', -1000);
        g.append('line').attr('class', 'end_right').attr('x1', -1000).attr('x2', -1000).attr('y1', -1000).attr('y2', -1000); // just accept it!
        g.append('rect').attr('class', 'marker');
        
        // errorLine
        svg.selectAll('.errorLine').attr('x1', function getX1(d) {
            var dPosition = x(d.data.value + d.data.std_error) - x(d.data.value);
            return x(d.data.value) - dPosition < 0 ? 0 : x(d.data.value) - dPosition;
        }).attr('y1', function getY1(d) {
            var fCurve = d.formula;
            return y(fCurve(d.data.value));
        }).attr('x2', function getX2(d) {
            return x(d.data.value + d.data.std_error) > width ? width : x(d.data.value + d.data.std_error);
        }).attr('y2', function getY2(d) {
            var fCurve = d.formula;
            return y(fCurve(d.data.value));
        });
        
        
        // rightLine
        svg.selectAll('.end_right').filter(function(d) {
            return x(d.data.value + d.data.std_error) < width;
        }).attr('x1', function getX1(d) {
            return x(d.data.value + d.data.std_error);
        }).attr('y1', function getY1(d) {
            // upper
            var fCurve = d.formula;
            return y(fCurve(d.data.value)) - iPxOffSetErrorBar;
        }).attr('x2', function getX2(d) {
            return x(d.data.value + d.data.std_error);
        }).attr('y2', function getY2(d) {
            var fCurve = d.formula;
            return y(fCurve(d.data.value)) + iPxOffSetErrorBar;
        });

        // leftLine
        svg.selectAll('.end_left').filter(function(d) {
            var dPosition = x(d.data.value + d.data.std_error) - x(d.data.value);
            return x(d.data.value) - dPosition > 0;
        }).attr('x1', function getX1(d) {
            var dPosition = x(d.data.value + d.data.std_error) - x(d.data.value);
            return x(d.data.value) - dPosition;
        }).attr('y1', function getY1(d) {
            var fCurve = d.formula;
            return y(fCurve(d.data.value)) - iPxOffSetErrorBar;
        }).attr('x2', function getX2(d) {
            var dPosition = x(d.data.value + d.data.std_error) - x(d.data.value);
            return x(d.data.value) - dPosition;
        }).attr('y2', function getY2(d) {
            var fCurve = d.formula;
            return y(fCurve(d.data.value)) + iPxOffSetErrorBar;
        });

        // marker
        svg.selectAll('.marker').attr('x', function getX(d) {
            return x(d.data.value) - 2.5;
        }).attr('y', function getY(d) {
            var fCurve = d.formula;
            return y(fCurve(d.data.value)) - 2.5;
        }).attr('width', '5').attr('height', '5').attr('fill', function(d) {
            return color(oColorMapping[d.curveId]);
        });

        // title
        svg.append('text').attr('class', 'Title').attr('x', margin.left).attr('y', -20).attr('text-anchor', 'start').text(this.title);

        // Symbols - color for different treatments, symbol type for different biological replicates
        for (var i = 0; i < aPoints.length; i++) {
            if (aPoints[i].length > 0) {
                svg.append('g').attr('class', 'dots').selectAll('.dot').data(aPoints[i]).enter().append('path').attr('class', 'point').attr('id', function() {
                    return 'dots-' + aProperties[i].curveId;
                }).attr('ModelId', function() {
                    return aProperties[i].curveId;
                }).attr('d', d3.symbol().type(d3.symbols[aProperties[i].curveId % 6])).attr('transform', function(d) {
                    return 'translate(' + x(d.x) + ',' + y(d.y) + ') scale(0.75)';
                }).on('mouseover', function() {
                    var sId = d3.select(this).attr('id').split('-')[1]; // next time: we do data binding correctly
                    that.highlightMouseOver(sId, x, y);
                }).on('mouseout', function() {
                    var sId = d3.select(this).attr('id').split('-')[1]; // next time: we do data binding correctly
                    that.unHighlightMouseOver(sId, x, y);
                }).style('fill', function() {
                    return color(oColorMapping[aProperties[i].curveId]);
                });
            }
        }

        svg.append('text').attr('class', 'x_label').attr('text-anchor', 'middle').attr('x', width / 2).attr('y', height + 30).text('Treatment [' + aProperties[0].doseUnit + ']');

        svg.append('text').attr('class', 'y_label').attr('transform', 'rotate(-90)').attr('y', -40).attr('x', 0 - height / 2).text(aProperties[0].responseUnit);

        // Legend
        var LegendLineWidth = 25;
        var LegendRowHeight = 10;
        var LegendOffset = 5;

        var legend = svg.selectAll('.legend').data(aProperties).enter().append('g').attr('id', function(d, i) {
            return 'CETSALegend-' + i;
        }).attr('class', 'legend').attr('ModelId', function(d) {
            return d.curveId;
        }).attr('transform', function(d, i) {
            var combined = LegendRowHeight + LegendOffset;
            var x = width + LegendOffset * 2;
            var y = combined * i + (height - aProperties.length * combined) / 2;
            return 'translate(' + x + ',' + y + ')';
        });

        legend.append('path').attr('class', 'point').attr('id', function(d, i) {
            return 'dotsLegend-' + aProperties[i].curveId;
        })
        .attr('d', function(d, i) {
            return d3.symbol().type(d3.symbols[aProperties[i].curveId % 6])();
        })
        .attr('transform', 'translate(' + LegendLineWidth / 2 + ',' + LegendRowHeight / 2 + ') scale(0.75)').style('fill', function(d) {
            return color(oColorMapping[d.curveId]);
        });

        legend.append('line').attr('x1', 0).attr('id', function(d) {
            return 'legendLine-' + d.curveId;
        }).attr('x2', LegendLineWidth).attr('y1', LegendRowHeight / 2).attr('y2', LegendRowHeight / 2).style('stroke', function(d) {
            return color(oColorMapping[d.curveId]);
        }).style('stroke-dasharray', function(d) {
            var x = d.biologicalReplicate - 1;
            return Math.max(1, x * 3) + ',' + x * 3;
        });

        legend.append('text').attr('x', LegendLineWidth + LegendOffset).attr('y', LegendRowHeight - 1).attr('class', 'legendText').on('mouseover', function() {
            var sId = d3.select(this).attr('id').split('-')[1]; // next time: we do data binding correctly
            that.highlightMouseOver(sId, x, y);
        }).on('mouseout', function() {
            var sId = d3.select(this).attr('id').split('-')[1]; // next time: we do data binding correctly
            that.unHighlightMouseOver(sId, x, y);
        }).attr('id', function(d, i) {
            return 'legendText-' + aProperties[i].curveId;
        }).text(function(d) {
            return d.TREATMENT;
        });
        /* TODO: convert this to Vue.js
        for (i = 0; i < aProperties.length; i++) {
            $('#CETSALegend-' + i).tipsy({
                gravity: 's',
                html: true,
                title: function title() {
                    var sTreatment = 'Inhibitor: ' + this.__data__.TREATMENT + '<br>';
                    var sBioRep = 'biological Replicate: ' + this.__data__.biologicalReplicate + '<br>';
                    var sTechRep = 'technical Replicate: ' + this.__data__.technicalReplicate;
                    return sTreatment + sBioRep + sTechRep;
                }
            });
        }

        for (i = 0; i < aParameters.length; i++) {
            $('#BoxTipsy-' + i).tipsy({
                gravity: 's',
                html: true,
                title: function title() {
                    var text1 = 'Treatment: ' + this.__data__[1].TREATMENT + '<br>';
		    var text2 = 'EC50: ' + this.__data__[0]['ED50, inflection'].value.toFixed(2);
		    var text3 = ' ' + aProperties[0].doseUnit; + '<br>';
                    return text1 + text2 + text3;
                }
            });
        }*/
     },
     init: function () {
        // create Curve Function
        var aParameters = this.curveParameters;
        var aDataPoints = this.dataPoints;

        var aProperties = this.properties;
        var aPoints = [];

        d3.select(this.$el).selectAll('svg').remove();

        if (aParameters) {
            // we need smallest value possible not zero and make it one dimension smaller
            aDataPoints = aDataPoints.map(function(curve) {
                var minimalXValue = d3.min(curve.filter(function(d) {
                    return d[0] > 0;
                }), function(d) {
                    return d[0];
                });
                minimalXValue = minimalXValue / 10;
                return curve.map(function(d) {
                    return d[0] === 0 ? [minimalXValue, d[1]] : d;
                });
            });
            // Series Array for multiple Curves
            var aSeries = [];
            // add Curve Datasets for each dataset
            for (var i = 0; i < aParameters.length; i++) {
                aSeries.push(this.calculateCurvePoints(aParameters[i], aDataPoints[i]));
            }
            // calculate Data Points
            if (aDataPoints && aDataPoints.length > 0) {
                for (i = 0; i < aDataPoints.length; i++) {
                    if (aDataPoints[i] && aDataPoints[i].length > 0) {
                        aPoints.push(aDataPoints[i].map(function(d) {
                            return {
                                x: d[0],
                                y: d[1]
                            };
                        }));
                    }
                }
            }
            // Draw Plot with fitted Curve and actual Data Points
            if (aProperties.length >= 1) {
                this.drawPlot(aSeries, aPoints, aProperties, aParameters);
            }
        }
      }
   },
  
  mounted: function() {
        this.init();
    }
}
</script>

<style>
@import './GenericLinePlot.css.prdb'
</style>

