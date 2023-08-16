<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
  
    let legende;
  
    onMount(() => {
      var largeurLeg = 640;
      var couleursLeg = d3.scaleLinear()
        .range(["#fde0dd", "#dd3497", "#49006a", "#4575b4", "#abd9e9"])
        .domain([largeurLeg, (largeurLeg / 4) * 3, largeurLeg / 2, (largeurLeg / 4), 0])
        .interpolate(d3.interpolateHcl);
  
      var svg = d3.select(legende)
        .append("svg")
        .attr("width", "100%")
        .attr("viewBox", "0 0 " + largeurLeg + " " + largeurLeg / 10);
  
      var barres = svg.selectAll(".bars")
        .data(d3.range(largeurLeg), d => d)
        .enter().append("rect")
        .attr("class", "bars")
        .attr("x", function(d, i) {
            return d;
        })
        .attr("y", 5)
        .attr("height", 35)
        .attr("width", 2)
        .attr("fill", function(d, i) {
            return couleursLeg(d);
        });
    });
  </script>
  
  <div id="legende" bind:this={legende}></div>
  