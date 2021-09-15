<template>
  <div id="pathwayContainer">
    <svg id="pathway" viewBox="0 0 2000 1200">
      <rect
        width="2000"
        height="1200"
        fill="none"
        style="stroke-width: 2; stroke: lightgray"
      />
      <!--         :viewBox="0 + ' ' + height / 4 + ' ' + width + ' ' + height">-->
      <defs>
        <marker
          id="activationMarker"
          viewBox="-0 -5 7 7"
          refX="3"
          refY="0"
          orient="auto"
          markerWidth="7"
          markerHeight="8"
          overflow="visible"
          preserveAspectRatio="none"
          pointer-events="none"
        >
          <path d="M 0,-2 L 4 ,0 L 0,2" fill="#999" stroke="none" />
        </marker>
        <marker
          id="otherInteractionMarker"
          viewBox="-0 -5 7 7"
          refX="3"
          refY="0"
          orient="auto"
          markerWidth="7"
          markerHeight="8"
          overflow="visible"
          preserveAspectRatio="none"
          pointer-events="none"
        >
          <path
            d="M 0,-1.5 L 3.5 ,0 L 0,1.5 Z"
            fill="white"
            stroke="#999"
            stroke-width="0.5pt"
          />
        </marker>
        <marker
          id="inhibitionMarker"
          viewBox="-0 -5 7 7"
          refX="0.55"
          refY="0"
          orient="auto"
          markerWidth="7"
          markerHeight="8"
          overflow="visible"
          preserveAspectRatio="none"
          pointer-events="none"
        >
          <path
            d="M 0,-2.5 L 1.5,-2.5 L 1.5,2.5 L 0,2.5"
            fill="#999"
            stroke="none"
          />
        </marker>
      </defs>
      <svg
        id="pathwayLegend"
        :width="legendWidth"
        :height="legendHeight"
        x="25"
        y="25"
      ></svg>
    </svg>
  </div>
</template>

<script>
import * as d3 from "d3";

export default {
  name: "Graph",
  props: {
    graphdataSkeleton: {
      type: Object,
      default: () => {
        return {
          nodes: undefined,
          links: undefined,
        };
      },
    },
    graphdataPTM: {
      type: Object,
      default: () => {
        return {
          nodes: undefined,
          links: undefined,
        };
      },
    },
    selectDownstreamButtonToggle: {
      type: Boolean,
      default: () => undefined,
    },
  },

  data() {
    return {
      nodeHeight: 10,
      ptmNodeWidth: 15,
      ptmNodeHeight: 10,
      nodes: undefined,
      links: undefined,
      groupIds: undefined,
      zoom: undefined,
      nUnselected: 0,
      legendHeight: 350,
      legendWidth: 450,
    };
  },
  watch: {
    //TODO: This might not be the best way to solve the 'rerender-graph-on-change' issue
    graphdataPTM: function () {
      this.computePTMGraph();
      this.renderGraph();

      this.onSelectedNodesChanged();
      d3.select("#pathway")
        .transition()
        .call(
          this.zoom.scaleTo,
          d3.zoomTransform(d3.select("#pathway").node()).k
        );
    },

    nUnselected: function () {
      this.$emit("update:isEverythingSelected", this.nUnselected === 0);
    },

    selectDownstreamButtonToggle: function () {
      /*
       * This is a bit hacky but the only way I could implement the emission of an event from parent to child
       * without coupling them very strongly. The parent event toggles this boolean prop, which triggers the watcher.
       * Note that the actual boolean value of the toggle variable is irrelevant, I just needed something that changed
       * so it can be watched.
       */
      d3.select("#nodeG")
        .selectAll("*")
        .filter((d) => d.selected)
        .each((d) => this.selectDownstreamNodes(d));
      this.onSelectedNodesChanged();
    },
  },
  mounted() {
    this.computeSkeletonGraph();
    const VERTICAL_SCALING_FACTOR = 1.5;
    this.nodes.forEach((node) => (node.y = node.y * VERTICAL_SCALING_FACTOR));
    this.renderGraph();
    this.enableZoomingAndPanning();
    this.renderLegend();
  },
  methods: {
    computeSkeletonGraph() {
      this.nodes = this.graphdataSkeleton.nodes.map((d) =>
        Object.create(d, {
          selected: { value: true, writable: true },
        })
      );
      this.links = this.graphdataSkeleton.links.map((d) => Object.create(d));
    },
    computePTMGraph() {
      //Remove all PTM nodes and links (by trimming them to the length of the skeleton)
      this.nodes = this.nodes.slice(0, this.graphdataSkeleton.nodes.length);
      this.links = this.links.slice(0, this.graphdataSkeleton.links.length);

      //Add the new PTM nodes and links - if there are any
      if (this.graphdataPTM.nodes) {
        this.nodes = this.nodes.concat(
          this.graphdataPTM.nodes.map((d) => Object.create(d))
        );
        this.links = this.links.concat(
          this.graphdataPTM.links.map((d) => Object.create(d))
        );
      }
    },
    renderGraph() {
      d3.select("#pathway").selectAll().remove();
      d3.select("#pathway").append("g").attr("id", "groupG");
      d3.select("#pathway").append("g").attr("id", "linkG");
      d3.select("#pathway").append("g").attr("id", "nodeG");
      this.addReferences();
      this.drawGraph();
      this.addAnimation();
      this.addTooltips();
      this.enableNodeSelection();
    },
    addReferences() {
      //1. Nodes
      this.nodes.forEach((node) => {
        // a) For all PTM peptides: add reference to protein node
        // This goes for both individual PTMs nodes and summary nodes
        if (node.type.includes("ptm")) {
          for (let i = 0; i < this.nodes.length; i++) {
            if (this.nodes[i].id === node.proteinId) {
              node.protein = this.nodes[i];
              node.x = node.protein.x;
              node.y = node.protein.y;
              break;
            }
          }
        }
        //b) For the PTM summary nodes: add references to individual PTM nodes
        if (
          node.type.includes("summary") &&
          ["up", "down"].includes(node.regulation)
        ) {
          node.ptmNodes = [];
          node.ptmIds.forEach((nodeId) => {
            this.nodes.forEach((otherNode) => {
              if (otherNode.id === nodeId) {
                node.ptmNodes.push(otherNode);
              }
            });
          });
        }
      });

      //2. For all links: add reference to source and target nodes
      this.links.forEach((link) => {
        for (let i = 0; i < this.nodes.length; i++) {
          if (this.nodes[i].id === link.sourceId) {
            link.source = this.nodes[i];
            break;
          }
        }
        for (let i = 0; i < this.nodes.length; i++) {
          if (this.nodes[i].id === link.targetId) {
            link.target = this.nodes[i];
            break;
          }
        }
      });
    },

    drawGraph() {
      const graph = this;
      const nodeG = d3.select("#nodeG");
      const linkG = d3.select("#linkG");
      const groupG = d3.select("#groupG");
      const nodesSvg = nodeG
        .selectAll("g")
        .data(graph.nodes, (d) => d.id)
        .join("g")
        .attr(
          "class",
          (d) => `node ${d.type} ${d.regulation ? d.regulation : ""}`
        );
      //TODO: Does it make sense to assign unique key?
      // .attr("key", d => d.id)

      //Draw each node as a rectangle
      nodesSvg
        .append("rect")
        .attr(
          "class",
          (d) => `node-rect ${d.type} ${d.regulation ? d.regulation : ""}`
        )
        .attr("rx", graph.nodeHeight)
        .attr("ry", graph.nodeHeight);

      //

      //Add labels to the nodes
      nodesSvg
        .append("text")
        .attr(
          "class",
          (d) => `node-label ${d.type} ${d.regulation ? d.regulation : ""} `
        );
      nodesSvg
        .select(".node-label")
        .text((d) => {
          if (d.label === null) return "";
          if (d.label.startsWith("TITLE:"))
            return d.label.substr(6).toUpperCase();
          if (d.type === "compound") return d.keggIds[0].split(":")[1];
          if (d.geneNames && d.geneNames.length > 1)
            return `${d.geneNames[0]}, ...`;
          else return d.label.split(/,|\//)[0];
        })
        .each(function (d) {
          let circleWidth = graph.nodeHeight * 2,
            textLength = this.getComputedTextLength(),
            textWidth = textLength + graph.nodeHeight;
          if (circleWidth > textWidth) {
            d.isCircle = true;
            d.rectX = -graph.nodeHeight;
            d.rectWidth = circleWidth;
          } else {
            d.isCircle = false;
            d.rectX = -(textLength + graph.nodeHeight) / 2;
            d.rectWidth = textWidth;
            d.textLength = textLength;
          }
        });

      //Position the nodes within their 'g' elements
      nodesSvg
        .select(".node-rect")
        .attr("x", (d) =>
          d.type === "ptm" ? -0.5 * graph.ptmNodeWidth : d.rectX
        )
        .attr("y", (d) =>
          d.type === "ptm" ? -0.5 * graph.ptmNodeHeight : -graph.nodeHeight
        )
        .attr("width", (d) =>
          d.type === "ptm" ? graph.ptmNodeWidth : d.rectWidth
        )
        .attr("height", (d) =>
          d.type === "ptm" ? graph.ptmNodeHeight : graph.nodeHeight * 2
        );

      //Logic for node groups (KEGG sometimes subsumes multiple nodes in a group)
      graph.groupIds = new Set(
        graph.nodes.filter((d) => d.groupId).map((d) => d.groupId)
      );
      groupG
        .selectAll(".group-path")
        .data(graph.groupIds)
        .join("g")
        .attr("class", "group-path")
        .append("path");

      //Draw links as lines with appropriate arrowheads
      linkG
        .selectAll("line")
        .data(graph.links)
        .join("line")
        .attr("class", (d) => `link ${d.type} ${d.subtypes.join(" ")}`)
        .attr("marker-end", (d) => {
          if (d.subtypes.includes("inhibition"))
            return "url(#inhibitionMarker)";
          else if (d.subtypes.includes("binding/association")) return "";
          else if (d.subtypes.includes("activation"))
            return "url(#activationMarker)";
          else return "url(#otherInteractionMarker)";
        })
        .attr("stroke-dasharray", (d) =>
          d.subtypes.includes("binding/association") ? "3 3" : undefined
        );

      //Add paths for the edgelabels
      linkG
        .selectAll(".edgepath")
        .data(graph.links)
        .join("path")
        .attr("class", "edgepath")
        .attr("fill-opacity", 0)
        .attr("stroke-opacity", 0)
        .attr("id", (d, i) => "edgepath" + i);

      //Add the actual edgelabels
      const edgelabels = linkG
        .selectAll(".edgelabel")
        .data(graph.links)
        .join("text")
        .attr("class", "edgelabel")
        .attr("font-size", 10)
        .attr("fill", "#4e4e4e");

      //Put the edgelabels onto the paths
      edgelabels
        .append("textPath")
        .attr("xlink:href", (d, i) => "#edgepath" + i)
        .attr("startOffset", "50%")
        .text((d) => {
          //Right now, the labels are limited to (de)phosphorylation
          if (d.subtypes.includes("phosphorylation")) return "+p";
          else if (d.subtypes.includes("dephosphorylation")) return "-p";
          else if (d.subtypes.includes("ubiquitination")) return "+u";
          else if (d.subtypes.includes("glycosylation")) return "+g";
          else if (d.subtypes.includes("methylation")) return "+m";
          else return "";
        });
    },
    addAnimation() {
      let graph = this;
      const PTM_LINK_FORCE_STRENGTH = 2,
        PTM_LINK_FORCE_DISTANCE_MULTIPLIER = 1.025,
        PTM_COLLISION_FORCE_RADIUS = 7;

      //Add forces to the PTM nodes and links
      const simulation = d3
        .forceSimulation()
        //Link Force
        .force(
          "link",
          d3
            .forceLink(this.links)
            .strength((link) =>
              link.type.includes("ptmlink") ? PTM_LINK_FORCE_STRENGTH : 0
            )
            .distance((link) =>
              link.type.includes("ptmlink")
                ? link.target.textLength * PTM_LINK_FORCE_DISTANCE_MULTIPLIER
                : 0
            )
        )
        //Collision repulsion Force
        .force(
          "collide",
          d3
            .forceCollide()
            .radius((node) =>
              node.type.includes("ptm") ? PTM_COLLISION_FORCE_RADIUS : 0
            )
        );
      //I used to have center force, many-body force and x/y positional force in here as well
      //But they messed things up quite a lot

      //Define drag behavior
      function dragNodes(simulation) {
        function dragstarted(event, node) {
          //Disable tooltip while dragging
          d3.select("#nodetooltip")
            .style("opacity", "0")
            .attr("is-dragging", "true");
          if (!event.active) simulation.alphaTarget(0.3).restart();
          node.fx = node.x;
          node.fy = node.y;
        }

        function dragged(event, node) {
          node.fx = event.x;
          node.fy = event.y;
        }

        function dragended(event, node) {
          if (!event.active) simulation.alphaTarget(0);
          node.fx = null;
          node.fy = null;
          d3.select("#nodetooltip").attr("is-dragging", "false");
        }

        return d3
          .drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended);
      }

      function dragGroups(simulation) {
        function dragstarted(event) {
          if (!event.active) simulation.alphaTarget(0.3).restart();
        }

        function dragged(event, groupId) {
          const nodes = d3.select("#nodeG").selectAll("g");
          nodes
            .filter((d) => d.groupId === groupId)
            .each((d) => {
              d.x += event.dx;
              d.y += event.dy;
            });
        }

        function dragended(event) {
          if (!event.active) simulation.alphaTarget(0.3).restart();
        }

        return d3
          .drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended);
      }

      //Apply drag behaviour to the nodes of the graph
      d3.select("#nodeG").selectAll("g").call(dragNodes(simulation));

      d3.select("#groupG").selectAll("g").call(dragGroups(simulation));

      //Define animation
      simulation.nodes(graph.nodes).on("tick", function () {
        graph.nodes.forEach(function (node) {
          if (node.isCircle) {
            node.leftX = node.rightX = node.x;
          } else {
            node.leftX = node.x - node.textLength / 2 + graph.nodeHeight / 2;
            node.rightX = node.x + node.textLength / 2 - graph.nodeHeight / 2;
          }
        });

        //Set new positions of the nodes
        d3.select("#nodeG")
          .selectAll("g")
          .attr("transform", function (node) {
            return "translate(" + node.x + "," + node.y + ")";
          });

        //Set new positions of the links
        //This mess makes the arrowheads end exactly at the rim of the target node.
        d3.select("#linkG")
          .selectAll("line")
          .each(function (link) {
            let sourceX, targetX, midX, dx, dy, angle;
            if (link.source.rightX < link.target.leftX) {
              sourceX = link.source.rightX;
              targetX = link.target.leftX;
            } else if (link.target.rightX < link.source.leftX) {
              targetX = link.target.rightX;
              sourceX = link.source.leftX;
            } else {
              midX = (link.source.x + link.target.x) / 2;
              if (midX > link.target.rightX) {
                midX = link.target.rightX;
              } else if (midX > link.source.rightX) {
                midX = link.source.rightX;
              } else if (midX < link.target.leftX) {
                midX = link.target.leftX;
              } else if (midX < link.source.leftX) {
                midX = link.source.leftX;
              }
              targetX = sourceX = midX;
            }
            dx = targetX - sourceX;
            dy = link.target.y - link.source.y;
            angle = Math.atan2(dx, dy);
            //'binding/association' links need an offset
            let targetOffset = link.subtypes.includes("binding/association")
              ? 0
              : 3.5;
            link.sourceX = sourceX + Math.sin(angle) * graph.nodeHeight;
            link.targetX =
              targetX - Math.sin(angle) * (graph.nodeHeight + targetOffset);
            link.sourceY = link.source.y + Math.cos(angle) * graph.nodeHeight;
            link.targetY =
              link.target.y -
              Math.cos(angle) * (graph.nodeHeight + targetOffset);
          })
          //Now the new start and end coordinates of the link have been calculated and can be set
          .attr("x1", (link) => link.sourceX)
          .attr("y1", (link) => link.sourceY)
          .attr("x2", (link) => link.targetX)
          .attr("y2", (link) => link.targetY);

        //Set the new positions of the edge paths
        d3.select("#linkG")
          .selectAll(".edgepath")
          .attr(
            "d",
            (edgepath) =>
              "M " +
              edgepath.source.x +
              " " +
              edgepath.source.y +
              " L " +
              edgepath.target.x +
              " " +
              edgepath.target.y
          );

        //Optionally rotate the edge labels, if they have been turned upside-down
        d3.select("#linkG")
          .selectAll(".edgelabel")
          .attr("transform", function (link) {
            if (link.target.x < link.source.x) {
              let bbox = this.getBBox();
              let rx = bbox.x + bbox.width / 2;
              let ry = bbox.y + bbox.height / 2;
              return "rotate(180 " + rx + " " + ry + ")";
            } else {
              return "rotate(0)";
            }
          });
        //Logic to update node groups
        let valueline = d3
          .line()
          .x((d) => d[0])
          .y((d) => d[1])
          .curve(d3.curveCatmullRomClosed);

        function polygonGenerator(id) {
          let nodeCoords = d3
            .select("#nodeG")
            .selectAll("g")
            .filter((d) => d.groupId === id)
            .data()
            .reduce(
              (res, d) =>
                res.concat([
                  [d.leftX, d.y - graph.nodeHeight / 2],
                  [d.leftX, d.y + graph.nodeHeight / 2],
                  [d.rightX, d.y - graph.nodeHeight / 2],
                  [d.rightX, d.y + graph.nodeHeight / 2],
                ]),
              []
            );
          return d3.polygonHull(nodeCoords);
        }

        graph.groupIds.forEach((id) => {
          let path = d3
            .select("#pathway")
            .selectAll(".group-path:not(.legend)")
            .filter((d) => d === id);
          let centroid;
          path
            .select("path")
            .attr("transform", "scale(1) translate(0,0)")
            .attr("d", (groupPath) => {
              let polygon = polygonGenerator(groupPath);
              centroid = d3.polygonCentroid(polygon);

              return centroid
                ? valueline(
                    polygon.map((point) => [
                      point[0] - centroid[0],
                      point[1] - centroid[1],
                    ])
                  )
                : null;
            });
          if (centroid) {
            d3.select(path.node()).attr(
              "transform",
              `translate(${centroid[0]},${centroid[1]}) scale(1.33)`
            );
          }
        });
      });
      return simulation;
    },

    enableZoomingAndPanning() {
      this.zoom = d3.zoom().scaleExtent([0.1, 25]).on("zoom", this.zoomed);

      //Set up zoom for the svg
      d3.select("#pathway").call(this.zoom);

      //Trigger initial zoom event
      d3.select("#pathway").transition().call(this.zoom.scaleBy, 1);

      //Translate the view so that it is not hidden behind the legend
      d3.select("#pathway")
        .transition()
        .duration(0)
        .call(this.zoom.translateBy, 500, 0);
    },

    zoomed({ transform }) {
      //Geometric zooms
      d3.select("#nodeG").attr("transform", transform);
      d3.select("#linkG").attr("transform", transform);
      d3.select("#groupG").attr("transform", transform);

      //Semantic zooms
      d3.select("#pathway")
        .selectAll(".edgelabel:not(.legend)")
        .attr("visibility", transform.k < 1 ? "hidden" : "visible");
      d3.select("#pathway")
        .selectAll(".ptm:not(.summary):not(.legend)")
        .attr("display", transform.k < 2 ? "none" : "block");
      d3.select("#pathway")
        .selectAll(".ptm.summary:not(.unregulated):not(.legend)")
        .attr("display", transform.k > 2 ? "none" : "block");
    },

    addTooltips() {
      let that = this;
      //Remove tooltip if present
      d3.select("#nodetooltip").remove();

      //(Re-)Add tooltip to graph
      const tooltip = d3
        .select("#pathwayContainer")
        .append("div")
        .attr("class", "tooltip")
        .attr("id", "nodetooltip")
        .attr("is-dragging", "false")
        .style("max-width", "600px")
        .style("opacity", "0"); //Hide the tooltip initially

      //Define functionality of mouse entering/moving inside/exiting something
      const mouseenter = function (e, node) {
        if (tooltip.attr("is-dragging") === "false") {
          tooltip.transition().duration(250).style("opacity", "1");
          if (d3.select(this).attr("class").includes("link")) {
            tooltip.html(
              `${node.type[0].toUpperCase() + node.type.slice(1)} (${
                node.subtypes
              })`
            );
          } else if (node.type === "ptm") {
            tooltip.html(that.getPTMTooltipText(node));
          } else if (node.type === "gene") {
            tooltip.html(node.geneNames.join(", "));
          } else if (d3.select(this).attr("class").includes("summary")) {
            let regulationLabel;
            switch (node.regulation) {
              case "up": {
                regulationLabel = "Upregulated";
                break;
              }
              case "down": {
                regulationLabel = "Downregulated";
                break;
              }
              case "unregulated": {
                regulationLabel = "Unregulated";
                break;
              }
            }
            tooltip.html(
              `${node.label} ${regulationLabel} ${
                +node.label > 1 ? "Peptides" : "Peptide"
              }`
            );
          } else {
            tooltip.html(node.label);
          }
        }
      };

      const mousemove = function (e) {
        tooltip
          //The offset is trial and error, I could not figure this out programmatically
          .style("top", e.pageY - 115 + "px")
          .style("left", e.pageX + 15 + "px");
      };

      const mouseleave = function () {
        tooltip.transition().duration(250).style("opacity", "0");
      };

      //Apply this funcionality to the nodes and links of the graph
      d3.select("#nodeG")
        .selectAll("g")
        .on("mouseenter", mouseenter)
        .on("mousemove", mousemove)
        .on("mouseleave", mouseleave);

      d3.select("#linkG")
        .selectAll("line")
        .on("mouseenter", mouseenter)
        .on("mousemove", mousemove)
        .on("mouseleave", mouseleave);
    },

    getPTMTooltipText(node) {
      return (
        "<pre style='text-align: left'>" +
        "Regulation:        " +
        node.regulation +
        "<br>" +
        "Modified sequence: " +
        node.modifiedSequence +
        "<br>" +
        "Gene Name(s):      " +
        node.geneName +
        "<br>" +
        "Uniprot ID(s):     " +
        node.uniprotId +
        "<br>" +
        "pEC50 =            " +
        node.logEC50 +
        "<br>" +
        "|Effect| =         " +
        node.absEffect +
        "<br>" +
        "R^2 =              " +
        node.r2 +
        "<br>" +
        "Experiment:        " +
        node.experiment +
        "<br>" +
        "</pre>"
      );
    },

    enableNodeSelection() {
      let that = this;
      //Handle selection of a node
      d3.select("#nodeG")
        .selectAll("g")
        .on("click", (e, d) => {
          //Unless the CTRL key is pressed, unselect everything first
          if (!e.ctrlKey)
            d3.select("#nodeG")
              .selectAll("g")
              .each((d) => (d.selected = false));
          //CTRL + Click on a selected node deselects the node, otherwise the node becomes selected
          let isSelected = !(e.ctrlKey && d.selected);
          //Apply this new value to the node itself and all attached ptm nodes
          d.selected = isSelected;
          d3.select("#pathway")
            .selectAll(".ptmlink:not(.legend)")
            .each((l) => {
              if (l.target === d) l.source.selected = isSelected;
            });
          //If the node is a PTM summary node, apply its selection status to its individual PTM nodes
          if (
            d.type.includes("summary") &&
            ["up", "down"].includes(d.regulation)
          ) {
            d.ptmNodes.forEach((d) => (d.selected = isSelected));
          }
          //If the node is a (non-summary) PTM node and it's a non-CTRL selection event,
          // emit an event to display the tooltip information permanently in the parent
          if (
            d.type.includes("ptm") &&
            !d.type.includes("summary") &&
            !e.ctrlKey
          ) {
            this.$emit("update-ptm-infobox", that.getPTMTooltipText(d));
          } else {
            this.$emit("update-ptm-infobox", undefined);
          }

          this.onSelectedNodesChanged();
          //Do not propagate event to canvas, because that would remove the highlighting
          e.stopPropagation();
        });

      //Handle click on canvas
      d3.select("#pathway").on("click", () => {
        //Remove all highlighting by setting every node to "selected"
        d3.select("#nodeG")
          .selectAll("g")
          .each((d) => (d.selected = true));
        this.$emit("update-ptm-infobox", undefined);
        this.onSelectedNodesChanged();
      });
    },

    //Helper method to recursively select all nodes downstream of a node
    selectDownstreamNodes(node) {
      //Select the node itself
      node.selected = true;
      //Select all PTM nodes attached to this node
      // (the node would be the target of the respective PTM link)
      d3.select("#pathway")
        .selectAll(".ptmlink:not(.legend)")
        .each((l) => {
          if (l.target === node) l.source.selected = true;
        });

      //Iterate over links and recurse on all target nodes of this node that have not yet been selected
      //The second criterion prevents endless recursion on circular paths
      d3.select("#linkG")
        .selectAll("line")
        .each((l) => {
          if (l.source === node && !l.target.selected) {
            this.selectDownstreamNodes(l.target);
          }
          //In the case of binding/association relations, we treat the edge as undirected,
          // i.e. the node could also be the target
          else if (
            l.subtypes.includes("binding/association") &&
            l.target === node &&
            !l.source.selected
          ) {
            this.selectDownstreamNodes(l.source);
          }
        });

      //After selecting downstream nodes, multiple PTMs are selected so the infobox needs to be disabled
      this.$emit("update-ptm-infobox", undefined);
    },

    onSelectedNodesChanged() {
      this.highlightSelectedNodes();
      this.sendSelectedPTMsToParent();
      this.updateNUnselected();
    },

    highlightSelectedNodes() {
      const opacityOfUnselected = 0.125;
      const opacityOfUnselectedPTM = 0.25;
      //Set opacity of all nodes which are selected to 1
      d3.select("#nodeG")
        .selectAll("g:not(.ptm)")
        .filter((node) => node.selected)
        .style("opacity", 1);

      //Reduce opacity of all nodes which are not selected (or hide them completely if they are PTM nodes)
      d3.select("#nodeG")
        .selectAll("g:not(.ptm)")
        .filter((node) => !node.selected)
        .style("opacity", opacityOfUnselected);

      //For the links, set the opacity to 1 only if both source and target are selected
      d3.select("#linkG")
        .selectAll("line")
        .filter((link) => link.source.selected && link.target.selected)
        .style("opacity", 1);

      d3.select("#linkG")
        .selectAll("line")
        .filter((link) => !(link.source.selected && link.target.selected))
        .style("opacity", opacityOfUnselected);

      d3.select("#pathway")
        .selectAll(".edgelabel:not(.legend)")
        .filter((link) => link.source.selected && link.target.selected)
        .style("opacity", 1);

      d3.select("#pathway")
        .selectAll(".edgelabel:not(.legend)")
        .filter((link) => !(link.source.selected && link.target.selected))
        .style("opacity", opacityOfUnselected);

      //PTM nodes might not yet have a 'selected' property - in that case, initialize them with that of their parent
      d3.select("#pathway")
        .selectAll(".ptm:not(.legend)")
        .filter((ptmNode) => {
          if (typeof ptmNode.selected === "undefined") {
            ptmNode.selected = ptmNode.protein.selected;
          }
          return ptmNode.selected;
        })
        .style("opacity", 1);

      d3.select("#pathway")
        .selectAll(".ptm:not(.legend)")
        .filter((ptmNode) => {
          if (typeof ptmNode.selected === "undefined") {
            ptmNode.selected = ptmNode.protein.selected;
          }
          return !ptmNode.selected;
        })
        .style("opacity", opacityOfUnselectedPTM);

      //Groups are selected if at least one of their members are selected
      const selectedGroupIDs = this.nodes
        .filter((d) => d.selected && d.groupId)
        .map((d) => d.groupId);

      d3.select("#groupG")
        .selectAll("g")
        .filter((d) => selectedGroupIDs.includes(d))
        .style("opacity", 1);

      d3.select("#groupG")
        .selectAll("g")
        .filter((d) => !selectedGroupIDs.includes(d))
        .style("opacity", opacityOfUnselected);
    },

    sendSelectedPTMsToParent() {
      const MAX_CURVES = 20;
      let selectedPTMNodes = this.nodes.filter(
        (d) => d.type === "ptm" && d.selected
      );

      let selectedPTMsString = selectedPTMNodes
        .map((d) => d.curveIds)
        .join(";");
      //If too many nodes are selected, plot no curves at all
      if ((selectedPTMsString.match(/;/g) || []).length >= MAX_CURVES) {
        selectedPTMsString = "";
      }
      this.$emit("ptm-selection-changed", selectedPTMsString);
    },

    updateNUnselected() {
      this.nUnselected = d3
        .select("#nodeG")
        .selectAll("g")
        .filter((d) => !d.selected)
        .size();
    },

    renderLegend() {
      var svg = d3.select("#pathwayLegend");

      //Bring legend to front of the canvas
      svg.node().parentNode.appendChild(svg.node());

      //Draw the frame
      svg
        .append("rect")
        .attr("x", 5)
        .attr("y", 5)
        .attr("width", this.legendWidth - 10)
        .attr("height", this.legendHeight - 10)
        .attr("fill", "white")
        .style("stroke-width", "1.5")
        .style("stroke", "darkgray");

      const xOffset = 25,
        yOffset = 40,
        lineHeight = 50,
        paragraphMargin = 15;

      svg
        .append("rect")
        .attr("class", "node-rect gene legend")
        .attr("x", xOffset)
        .attr("y", yOffset + lineHeight * 0 + paragraphMargin * 0)
        .attr("rx", 10)
        .attr("ry", 30)
        .attr("width", 35)
        .attr("height", 25);
      svg
        .append("text")
        .attr("class", "legend")
        .text("Gene/Compound")
        .attr("x", xOffset + 40)
        .attr("y", yOffset + lineHeight * 0 + paragraphMargin * 0);

      svg
        .append("rect")
        .attr("class", "node-rect map legend")
        .attr("x", xOffset + 250)
        .attr("y", yOffset + lineHeight * 0 + paragraphMargin * 0)
        .attr("rx", 10)
        .attr("ry", 30)
        .attr("width", 35)
        .attr("height", 25);
      svg
        .append("text")
        .attr("class", "legend")
        .text("Pathway")
        .attr("x", xOffset + 290)
        .attr("y", yOffset + lineHeight * 0 + paragraphMargin * 0);

      svg
        .append("rect")
        .attr("class", "node-rect group-path legend")
        .attr("x", xOffset)
        .attr("y", yOffset + lineHeight * 1 + paragraphMargin * 0)
        .attr("rx", 10)
        .attr("ry", 30)
        .attr("width", 35)
        .attr("height", 25);
      svg
        .append("text")
        .attr("class", "legend")
        .text("Group")
        .attr("x", xOffset + 40)
        .attr("y", yOffset + lineHeight * 1 + paragraphMargin * 0);

      svg
        .append("line")
        .attr("class", "link legend")
        .attr("x1", xOffset)
        .attr("y1", yOffset + lineHeight * 2 + paragraphMargin * 1)
        .attr("x2", xOffset + 65)
        .attr("y2", yOffset + lineHeight * 2 + paragraphMargin * 1)
        .attr("marker-end", "url(#activationMarker)");
      svg
        .append("text")
        .attr("class", "legend")
        .text("Activation")
        .attr("x", xOffset + 80)
        .attr("y", yOffset + lineHeight * 2 + paragraphMargin * 1);

      svg
        .append("line")
        .attr("class", "link legend")
        .attr("x1", xOffset + 200)
        .attr("y1", yOffset + lineHeight * 2 + paragraphMargin * 1)
        .attr("x2", xOffset + 265)
        .attr("y2", yOffset + lineHeight * 2 + paragraphMargin * 1)
        .attr("marker-end", "url(#inhibitionMarker)");
      svg
        .append("text")
        .attr("class", "legend")
        .text("Inhibition")
        .attr("x", xOffset + 280)
        .attr("y", yOffset + lineHeight * 2 + paragraphMargin * 1);

      svg
        .append("line")
        .attr("class", "link legend")
        .attr("x1", xOffset)
        .attr("y1", yOffset + lineHeight * 3 + paragraphMargin * 1)
        .attr("x2", xOffset + 65)
        .attr("y2", yOffset + lineHeight * 3 + paragraphMargin * 1)
        .attr("marker-end", "url(#otherInteractionMarker)");
      svg
        .append("text")
        .attr("class", "legend")
        .text("Other Interaction")
        .attr("x", xOffset + 80)
        .attr("y", yOffset + lineHeight * 3 + paragraphMargin * 1);

      svg
        .append("line")
        .attr("class", "link legend")
        .attr("x1", xOffset)
        .attr("y1", yOffset + lineHeight * 4 + paragraphMargin * 1)
        .attr("x2", xOffset + 65)
        .attr("y2", yOffset + lineHeight * 4 + paragraphMargin * 1)
        .attr("stroke-dasharray", "10 10")
        .style("stroke-width", 10);
      svg
        .append("text")
        .attr("class", "legend")
        .text("Binding/Association")
        .attr("x", xOffset + 80)
        .attr("y", yOffset + lineHeight * 4 + paragraphMargin * 1);

      svg
        .append("rect")
        .attr("class", "node-rect ptm up legend")
        .attr("x", xOffset)
        .attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2)
        .attr("rx", 25)
        .attr("ry", 25)
        .attr("width", 25)
        .attr("height", 25);

      // svg.append("text").attr("class", "legend")
      //     .text("/")
      //     .style("font-size", "26pt")
      //     .attr("x", xOffset + 28).attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2 + 1);

      svg
        .append("text")
        .attr("class", "legend")
        .text("Up-")
        .attr("x", xOffset + 30)
        .attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2);

      svg
        .append("rect")
        .attr("class", "node-rect ptm down legend")
        .attr("x", xOffset + 85)
        .attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2)
        .attr("rx", 25)
        .attr("ry", 25)
        .attr("width", 25)
        .attr("height", 25);

      // svg.append("text").attr("class", "legend")
      //     .text("/")
      //     .style("font-size", "26pt")
      //     .attr("x", xOffset + 68).attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2 + 1);

      svg
        .append("text")
        .attr("class", "legend")
        .text("Down-")
        .attr("x", xOffset + 115)
        .attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2);

      svg
        .append("rect")
        .attr("class", "node-rect ptm unregulated legend")
        .attr("x", xOffset + 200)
        .attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2)
        .attr("rx", 25)
        .attr("ry", 25)
        .attr("width", 25)
        .attr("height", 25);

      svg
        .append("text")
        .attr("class", "legend")
        .text("Not regulated")
        .attr("x", xOffset + 230)
        .attr("y", yOffset + lineHeight * 5 + paragraphMargin * 2);
    },
  },
};
</script>

<style lang="scss">
.link {
  stroke: #999;
  stroke-opacity: 0.6;
  stroke-width: 3;

  &.ptmlink {
    visibility: hidden;
  }

  &.maplink {
    stroke-dasharray: "5 2";
  }
}

.node {
  pointer-events: fill;
}

.node-rect {
  stroke: black;
  stroke-width: 1.5;
  cursor: move;

  &.map {
    fill: #89b9ce;
  }

  //&.gene {
  //  fill: #f3f3f3;
  //}

  &.gene,
  &.compound,
  &.ortholog {
    fill: #ececec;
  }

  &.group {
    opacity: 0.25;
  }

  &.ptm {
    opacity: 1;

    &.down {
      fill: #252bff;
    }

    &.up {
      fill: #ea0000;
    }

    &.unregulated {
      fill: #adadad;
    }
  }
}

.legend {
  pointer-events: none;
  dominant-baseline: central;
  font-size: 24px;
  stroke-width: 2.5;

  &.link {
    stroke-width: 5px;
  }

  &.node-rect {
    transform: translate(0px, -15px);
  }

  &.group-path {
    stroke-width: 3px;
  }
}

.node-label {
  stroke-width: 0;
  font-family: "Arial", serif;
  font-size: 8px;
  font-weight: normal;
  text-anchor: middle;
  dominant-baseline: middle;
  pointer-events: none;
  color: black;

  &.ptm.summary {
    font-size: 12px;
    font-weight: bold;
  }

  &.ptm {
    font-size: 7px;
  }
}

.tooltip {
  position: absolute;
  text-align: center;
  background-color: white;
  border: solid;
  border-width: 1px;
  border-radius: 5px;
  padding: 10px;
  pointer-events: none;
}

.group-path {
  fill: #60bdbd;
  stroke-width: 1px;
  stroke: #468989;
}
</style>