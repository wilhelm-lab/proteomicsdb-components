<template>
  <v-row class="bodymapDiv" id="bodymapDiv" justify="center">
    <v-btn-toggle v-if="selectedOrganism.taxcode === 9606"
                  v-model="selectedGender"
                  tile
                  :color="buttonsColor"
                  group
                  fixed top left
                  mandatory
                  style="margin-left: 20px; margin-right: -40px;"
                  >
                  <v-btn icon value="female">
                    <v-icon>fas fa-venus</v-icon>
                  </v-btn>
                  <v-btn icon value="male">
                    <v-icon>fas fa-mars</v-icon>
                  </v-btn>
    </v-btn-toggle>
    <div id="bodymap"
         class="bodymap"
         >
    </div>
      <div id="bodymapLegendDiv"
           class="bodymapLegendDiv"
           >
      </div>
  </v-row>
</template>

<script>
const d3 = require('d3');
import d3tip from 'd3-tip';
d3.tip = d3tip;
export default {
  components: {
  },
  props: {
    width: {
      type: String,
      default: '40%'
    },
    height: {
      type: String,
      default: '40%'
    },
    minWidthCSS: {
      type: [String, Number],
      default: '100'
    },
    minHeightCSS: {
      type: [String, Number],
      default: '100'
    },
    minWidth: {
      type: Number,
      default: 500
    },
    buttonsColor: {
      type: String,
      default: '#0065BD'
    },
    minHeight: {
      type: Number,
      default: 500
    },
    data: {
      type: Array,
      default: () => []
    },
    selectedOrganism: {
      type: Object,
      default: () => {}
    },
    drawOnMount: {
      type: Boolean,
      default: false
    }
  },
  data: function () {
    return {
      bodyMapSvg: '',
      svg: null,
      promiseLoadSvg: null,
      tip: d3.tip().attr('class', 'd3-tip').offset([0, 0]).html(() => {return("bla bla")}),
      selectedGender: 'female'
    }
  },
  watch: {
    data: function(newData) {
      if (newData && this.selectedOrganism) {
        this.redraw();
      }
    },
    selectedOrganism: function (oOrg) {
      if(this.selectedOrganism.taxcode !== oOrg.taxcode) {
        this.selectedGender = oOrg.taxcode === 9606 ? 'female' : null;
      }
    },
    selectedGender: function () {
      if (this.data && this.selectedOrganism) {
        this.redraw();
      }
    }
  },
  methods: {
    drawBodymap: function() {
      
      var that = this;
      that.svgLoaded = false;

      switch (this.selectedOrganism.taxcode) {
        case 9606:
        if(that.selectedGender === 'male') {
          d3.xml('/assets/bodyMap/man_new_with_IDs_final.svg').then((response) => { 
            d3.select(".bodymap").node().append(response.documentElement);
            that.svgLoaded = true;
            if (that.data) { that.bind(); }
          });
        } else if (that.selectedGender === 'female') {
          d3.xml('/assets/bodyMap/woman_new_with_IDs_final.svg').then((response) => { 
            d3.select(".bodymap").node().append(response.documentElement);
            that.svgLoaded = true;
            if (that.data) { that.bind(); }
          });
        }
        break;
        case 3702:
        d3.xml('/assets/bodyMap/AraTh_body_map_with_IDS.svg').then((response) => { 
          d3.select(".bodymap").node().append(response.documentElement)
          that.svgLoaded = true;
          if (that.data) {
            that.bind();
          }
        });
        break;
        case 10090:
        d3.xml('/assets/bodyMap/Mouse_body_map_with_IDS.svg').then((response) => { 
          d3.select(".bodymap").node().append(response.documentElement)
          that.svgLoaded = true;
          if (that.data) {
            that.bind();
          }
        });
        break; 
      }

      d3.select('.bodymap').select('path').call(this.tip);

      this.$emit('resetSelections', null);

    },
    getOrgans: function() {
      switch (this.selectedOrganism.taxcode) {
        case 9606:
        return {
          small_intestine: 'small_intestine',
          rectum: 'rectum',
          anus: 'anus',
          nasopharynx: 'nasopharynx',
          kidney: 'kidney',
          lung: 'lung',
          heart: 'heart',
          brain: 'brain',
          breast: 'breast',
          thyroid: 'thyroid',
          adrenal: 'adrenal',
          pancreas: 'pancreas',
          femur: 'femur',
          lymph_node: 'lymph_node',
          skeletal_muscle: 'skeletal_muscle',
          adipose: 'adipose',
          oral_cavity: 'oral_cavity',
          tonsils: 'tonsils',
          stomach: 'stomach',
          cardia: 'cardia',
          duodenum: 'duodenum',
          esophagus: 'esophagus',
          appendix: 'appendix',
          urinary_bladder: 'urinary_bladder',
          ovary: 'ovary',
          liver: 'liver',
          gall_bladder: 'gall_bladder',
          colon: 'colon',
          uterus: 'uterus',
          cervix_uterine: 'cervix_uterine',
          vagina: 'vagina',
          bone_marrow: 'bone_marrow',
          placenta: 'placenta',
          skin: 'skin',
          circulatory_system: 'circulatory_system',
          prostate: 'prostate',
          testis: 'testis',
          salivary_gland: 'salivary_gland'
        };
        case 10090:
        return {
          bto_0000047: 'adrenal gland',
          bto_0000141: 'bone marrow',
          bto_0000142: 'brain',
          bto_0000156: 'brown adipose tissue',
          bto_0000214: 'cell culture',
          bto_0000232: 'cerebellum',
          bto_0000243: 'vagina',
          bto_0000293: 'occipital lobe',
          bto_0000365: 'duodenum',
          bto_0000379: 'embryo',
          bto_0000408: 'epididymis',
          bto_0000439: 'eye',
          bto_0000484: 'frontal lobe',
          bto_0000562: 'heart',
          bto_0000601: 'hippocampus',
          bto_0000651: 'small intestine',
          bto_0000671: 'kidney',
          bto_0000672: 'hindbrain',
          bto_0000706: 'large intestine',
          bto_0000759: 'liver',
          bto_0000763: 'lung',
          bto_0000784: 'lymph node',
          bto_0000817: 'mammary gland',
          bto_0000959: 'esophagus',
          bto_0000961: 'olfactory bulb',
          bto_0000975: 'ovary',
          bto_0000988: 'pancreas',
          bto_0001103: 'skeletal muscle',
          bto_0001129: 'prostate gland',
          bto_0001203: 'salivary gland',
          bto_0001234: 'seminal vesicle',
          bto_0001253: 'skin',
          bto_0001279: 'spinal cord',
          bto_0001281: 'spleen',
          bto_0001284: 'femur',
          bto_0001307: 'stomach',
          bto_0001355: 'temporal lobe',
          bto_0001363: 'testis',
          bto_0001374: 'thymus',
          bto_0001379: 'thyroid gland',
          bto_0001385: 'tongue',
          bto_0001388: 'trachea',
          bto_0001418: 'urinary bladder',
          bto_0001424: 'uterus',
          bto_0001456: 'white adipose tissue'
        };
        case 3702:
        return {po_0009046: 'flower',
          po_0009001: 'fruit',
          po_0025268: 'fruit_septum',
          po_0000033: 'fruit_valve',
          po_0009010: 'seed',
          po_0009031: 'sepal',
          po_0009032: 'petal',
          po_0009029: 'stamen',
          po_0009030: 'carpel',
          po_0000013: 'cauline_leaf',
          po_0005005: 'shoot_axis_internode',
          po_0005004: 'shoot_axis_node',
          po_0000014: 'rosette_leaf',
          po_0009005: 'root',
          po_0020030: 'cotyledon',
          po_0006079: 'shoot_system_meristem',
          po_0025291: 'seedling_hypocotyl',
          po_0020135: 'root_differential_zone',
          po_0000025: 'root_tip',
          po_0030112: 'pedicel',
          // po_0025281: 'pollen',
        po_0025280: 'pollen'};
        default:
        return [];
      }
    },

    reset: function() {
      var that = this;
      var svg = d3.select('.bodymap').selectAll('svg');
      if (typeof this.selectedOrganism !== 'undefined') {
        Object.entries(that.getOrgans()).forEach(function(organ) {
          var organ_name = organ[1];
          var svgOrgan = svg.select('#' + organ_name.toLowerCase());
          // reset organs
          svgOrgan.attr('fill', '#808080')
          .attr('fill-opacity', 0.2)
          .attr('stroke-width', 1)
          .attr('stroke', 'none');
        });
      }
    },

    bind: function() {
      var that = this;

      var red = '#FF0000';
      var green = '#00FF00';
      var minIntensity = d3.min(this.data, function(d) {
        return +d.NORMALIZED_INTENSITY;
      });
      var maxIntensity = d3.max(this.data, function(d) {
        return +d.NORMALIZED_INTENSITY;
      });
      that.intensityToColor = d3.scaleLinear()
      .domain([minIntensity, maxIntensity])
      .interpolate(d3.interpolateRgb)
      .range([green, red]);

      this.reset();

      if (this.data.length > 0) {
        var color_data = [];

        Object.entries(that.getOrgans()).forEach(function(organ) {
          var organ_id = organ[0];
          var organ_name = organ[1];
          var svgOrgan = d3.select('#'+organ_id);
          svgOrgan.style("visibility", "hidden");

          // fill organs with sap_synonym with color
          if (that.data) {
            var related_tissues = that.data.filter(function(element) {
              if (element.SAP_SYNONYM) {
                return element.SAP_SYNONYM.replace(':', '_').toLowerCase() == organ_id;
              }
              return false;
            });

            if (related_tissues.length > 0) { // tissues found that match sap_synonym!
              var max_intensity;
              max_intensity = d3.max(related_tissues, function(d) {
                return d.NORMALIZED_INTENSITY;
              });

              var organ_color = that.intensityToColor(max_intensity);
              color_data.push({
                color: organ_color,
                intensity: max_intensity,
                sap_synonym: organ_name
              });

              //var label = 'maximum intensity';
              // treat organ with many parts as one
              svgOrgan.attr('fill', organ_color)
              .attr('stroke', '#787878')
              .on("mouseenter", function () {

                d3.select(this).style("cursor", "pointer");
                //that.tip.html(function () {
                  //  return "bla bla bla"
                //});

                // show the tooltip
                //that.tip.direction("n");
                //that.tip.offset([0,0]);

                //that.tip.show(t, svgOrgan.node());
              });
              // .tipsy({
                //   gravity: 's',
                //   html: true
              // })
              // .attr('original-title', organ_name.replace('_', ' ') + '<hr>' + label + ': ' + max_intensity.toFixed(2));

              // while (svgOrgan.children().length > 0) {
                //   svgOrgan.attr('fill', organ_color)
                //   .attr('stroke', '#787878')
                //   .tipsy({
                  //     gravity: 's',
                  //     html: true
                //   })
                //   .attr('original-title', organ_name.replace('_', ' ') + '<hr>' + label + ': ' + max_intensity.toFixed(2));
                //   svgOrgan = svgOrgan.children();
              // }
            }
          }
        });

        // bind click
        that.data.forEach(function(d) {
          if (d.SAP_SYNONYM) { // bind only organs for which the synonyms are in the data
            var svgOrgan = d3.select('#' + d.SAP_SYNONYM.replace(':', '_').toLowerCase());

            svgOrgan.style("visibility", "visible");
            svgOrgan.attr("class", 'unselected')
            .attr("display", 'inherit')
            .attr("fill-opacity", 0.2)
            .attr("stroke", '#787878')
            .on("click", function() {
              that.selectOrgan(d.SAP_SYNONYM);
            });
          }
        });
        color_data.sort(function(a, b) { // due to avaraging colors may not be sorted
          if (a.intensity > b.intensity) {
            return -1;
          }
          if (a.intensity < b.intensity) {
            return 1;
          }
          return 0;
        });
        that.updateLegend(color_data);
      }
    },
    selectOrgan: function(d) {
      this.toggleOrgan(d);
      this.$emit('organSelected', d);

    },
    toggleOrgan: function (organName) {
      var svgOrgan = d3.select('#'+organName.replace(':','_').toLowerCase());
      if (svgOrgan.node() !== null) {
        if (svgOrgan.attr('class') === 'selected') {
          svgOrgan.attr('class', 'unselected')
          .attr('display', 'inherit')
          .attr('fill-opacity', 0.2)
          .attr('stroke', '#787878')
          ;
          return false;
        }
        svgOrgan.attr('class', 'selected')
        .attr('fill-opacity', 0.7)
        .attr('stroke', 'black');
        return true;
      }

    },
    updateLegend: function(data) {
      var height = 360;
      var margin_top = 35;

      var svg = d3.select('#bodymapLegendDiv').select('svg');
      svg.selectAll('g').remove();

      var groups = svg.selectAll('g')
      .data(data)
      .enter()
      .append('g');

      groups.append('rect')
      .attr('width', 10)
      .attr('height', height / data.length - 1)
      .attr('y', function(d, i) {
        return i * (height / data.length) + margin_top;
      })
      .attr('x', 7)
      .attr('fill', function(d) {
        return d.color;
      });

      svg.append('g')
      .attr('transform', 'translate(0,20)')
      .append('text')
      .text('high')
      .attr('font-weight', 'bold')
      .attr('font-size', '12px');

      svg.append('g')
      .attr('transform', 'translate(3,420)')
      .append('text')
      .text('low')
      .attr('font-weight', 'bold')
      .attr('font-size', '12px');

      //      var label = 'Intensity';

      // tipsy for legend
      //$('#' + this.sId + ' .sapProteomicsdbBodymapLegend svg rect').each(function(d, i) {
        //  $(this).tipsy({
          //    gravity: 'w',
          //    html: true,
          //    title: function() {
            //      return label + ': ' + this.__data__.intensity.toFixed(2);
          //    }
        //  });
      //});
    },
    drawBodymapLegend: function() {
      var height = 720;
      d3.select(this.$el).select('.bodymapLegendDiv').append('svg')
      .attr('width', 70)
      .attr('height', height);
    },
    redraw: function(){
      d3.select('.bodymapDiv').selectAll('svg').remove();
      this.drawBodymap();
      this.drawBodymapLegend();
    },
    getSVG: function () {
      return d3.select(this.$el).selectAll('svg').node();
    }
  },
  mounted: function () {
//    if ( this.drawOnMount && this.data) {
//      this.redraw();
//    }
  }
}
</script>

<style scss scoped>

/* probably won't work tooltip */
.d3-tip {
  line-height: 1;
  font-weight: bold;
  padding: 12px;
  background: rgba(0, 0, 0, 0.8);
  color: #fff;
  border-radius: 2px;
  pointer-events: none;
}

/* Creates a small triangle extender for the tooltip */
.d3-tip:after {
  box-sizing: border-box;
  display: inline;
  font-size: 10px;
  width: 100%;
  line-height: 1;
  color: rgba(0, 0, 0, 0.8);
  content: "\25BC";
  position: absolute;
  text-align: center;
  pointer-events:none !imporatant;
}

/* Style northward tooltips differently */
.d3-tip.n:after {
  margin: -1px 0 0 0;
  top: 100%;
  left: 0;
  pointer-events:none !imporatant;
}
.d3-tip,
.d3-tip n,
.d3-tip e,
.d3-tip s
{
  z-index: 2000;
}
.v-btn-toggle {
  flex-direction: column;
}
</style>

