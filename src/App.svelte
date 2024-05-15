<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let data = [];
  let summaryByYear = [];
  let allDepartments = [];
  let selectedDepartments = new Set();
  let colorScale = d3.scaleOrdinal(d3.schemeTableau10); // Color scale for departments

  onMount(() => {
    d3.csv('/capes_data.csv').then(loadedData => {
      data = loadedData;
      processData();
      drawGraph();
    }).catch(error => {
      console.error("Failed to load CSV:", error);
    });
  });

  function processData() {
    data.forEach(d => {
      const parts = d.Course.split(" ");
      d.Department = parts[0];
      d.Year = d.Quarter.slice(-2);
    });

    const groupByYear = d3.rollup(data, v => d3.mean(v, d => d['Study Hours per Week']), d => d.Year);
    summaryByYear = Array.from(groupByYear, ([Year, AverageStudyHours]) => ({ Year, AverageStudyHours }));
    allDepartments = Array.from(new Set(data.map(d => d.Department)));
  }

  function drawGraph() {
    const svg = d3.select('#chart');
    svg.selectAll("*").remove();

    const maxStudyHours = d3.max([...summaryByYear.map(d => d.AverageStudyHours), ...Array.from(selectedDepartments).flatMap(dept =>
      data.filter(d => d.Department === dept).map(d => d['Study Hours per Week'])
    )]);

    const margin = { top: 20, right: 30, bottom: 30, left: 50 };
    const width = 800 - margin.left - margin.right;
    const height = 400 - margin.top - margin.bottom;

    const xScale = d3.scaleLinear()
      .domain(d3.extent(summaryByYear, d => +d.Year))
      .range([0, width]);
    const yScale = d3.scaleLinear()
      .domain([0, maxStudyHours])
      .range([height, 0]);

    const g = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);
    g.append('g').attr('transform', `translate(0,${height})`).call(d3.axisBottom(xScale).tickFormat(d3.format("20")));
    g.append('g').call(d3.axisLeft(yScale));

    const line = d3.line().x(d => xScale(+d.Year)).y(d => yScale(d.AverageStudyHours));
    g.append('path').datum(summaryByYear).attr('fill', 'none').attr('stroke', 'steelblue').attr('stroke-width', 2).attr('d', line);

    selectedDepartments.forEach(dept => {
      const deptData = data.filter(d => d.Department === dept);
      const deptSummary = d3.rollup(deptData, v => d3.mean(v, d => d['Study Hours per Week']), d => d.Year);
      const deptSummaryArray = Array.from(deptSummary, ([Year, AverageStudyHours]) => ({ Year, AverageStudyHours }));

      g.append('path').datum(deptSummaryArray)
        .attr('fill', 'none')
        .attr('stroke', colorScale(dept))
        .attr('stroke-width', 2)
        .attr('d', line);
    });
  }

function toggleDepartment(dept) {
    const button = document.querySelector(`button[data-dept="${dept}"]`);
    if (selectedDepartments.has(dept)) {
        selectedDepartments.delete(dept);
        button.style.color = "black";
        button.style.fontWeight = "normal";
    } else {
        selectedDepartments.add(dept);
        button.style.color = colorScale(dept);
        button.style.fontWeight = "bold";
    }
    drawGraph();
}


</script>

<main>
    <h1>Study Hours Analysis</h1>
	<i>Welcome to our webpage dedicated to studying habits at UC San Diego. Did you know that, on average, students at UCSD dedicate approximately 5.5 hours per week to studying for each class they take?

	Curious about how your own major compares? Check out our interactive analysis where you can explore the study habits across different departments. Click on the buttons representing various majors to see how their study hours stack up against each other, and the campus average. </i>

	<div>
        {#each allDepartments as dept}
        <button data-dept="{dept}" on:click={() => toggleDepartment(dept)} style="color: {selectedDepartments.has(dept) ? colorScale(dept) : 'black'}; border-color: {selectedDepartments.has(dept) ? colorScale(dept) : '#ccc'};">
            {dept}
        </button>
        {/each}
    </div>
    <svg id="chart"></svg>
</main>


<style>
  main {
    text-align: center;
    font-family: Arial, sans-serif;
  }
  .button-container {
    margin: 20px 0;
    display: inline-block;
  }
  button {
    margin: 2px;
    border: 2px solid #ccc;
    background: none;
    cursor: pointer;
    padding: 5px 10px;
    transition: all 0.3s;
  }
  button:hover {
    background-color: #f4f4f4;
    border-color: #888;
  }
  button:active {
    background-color: #f8f8f8;
    border-color: #888;
  }
  svg {
    display: block;
    margin: 0 auto;
    width: 800px;
    height: 400px;
  }
  div {
	line-height: 150%;
	margin: 30pt;
  }
</style>
