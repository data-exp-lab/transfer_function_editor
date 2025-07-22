<template>  
    <div>
      <h1 class="green">TF Editor</h1>
      <p>single-click: select point closest to click </p>
      <p>double-click: add a new point (and select it)</p>
      <p>click+move+release: drag the selected point </p>
    </div>    
    <div ref="svg-container"
         @mousedown="mouseDown($event)" 
         @mouseup="mouseUp($event)"
         style="background-color:#ededed; 
                width: 800px; 
                height:500px">        
      <svg ref="mySVG"></svg>
    </div>
    <div>
      <button @click="exportTF()">export TF</button>
    </div>
  </template>


<script lang="ts">

import { onMounted, ref, type Ref, type TemplateRef, useTemplateRef} from 'vue'
import * as d3 from 'd3';
import type { Path } from 'typescript';


interface svg_context_types {
  'xScale': d3.ScaleLinear<number, number, never>,
  'yScale': d3.ScaleLinear<number, number, never>,
  'width': number, 
  'height': number, 
  'path': SVGPathElement | null,
}

export default {
    name: "TransferFunctionEditor",

    setup(){

        const mySVG = ref(null)
        const svgContainer: TemplateRef<HTMLDivElement> = useTemplateRef('svg-container');
        const downClickTimes: number[] = [0., 0.,]        
        const doubleClickTime_ms = 500
        const clickDownPosition: number[] = [0., 0.]        
        const selectedPoint: Ref<number|null> = ref(null)
        const selectedPointID: Ref<number|null> = ref(null)
        const maxPointId: Ref<number> = ref(0)
        const mouseEventType = ref('double-click')
        const svg_context: Ref<svg_context_types|null> = ref(null)

        // Sample data
        const data = [
                    { x: 0, y: 0, id:0 },                    
                    { x: 2, y: 0, id:1},                                        
                    { x: 4, y: 0, id:2 },                    
                    { x: 6, y: 0, id:3 },                    
                    { x: 8, y: 0, id:4},                    
                    { x: 10, y: 0, id:5 },
                    ];

        function exportTF(){
          console.log("try to interpolate") 
          if (svg_context.value){                   
            const path = svg_context.value.path 
            const xScale = svg_context.value?.xScale
            const yScale = svg_context.value?.yScale
            
            if (path && xScale && yScale){
              console.log('svg path total length:', path.getTotalLength())
              const n_path = 100
              const d_path = path.getTotalLength() / n_path
              const x_pts = [] 
              const y_pts = []
              for (let ipt = 0; ipt < n_path; ipt++){
                const p = path.getPointAtLength(ipt * d_path)
                x_pts.push(xScale.invert(p.x))
                y_pts.push(yScale.invert(p.y))
              }
              // save this somewhere
              console.log('interpolated x, y', x_pts, y_pts)
            }
          }          
        }

        function buildPlot(){
            const svgCval = svgContainer.value;
            if (svgCval){
              const width_str: string = svgCval.style.width;
              const height_str: string = svgCval.style.height;
              const width=Number(width_str.replace('px',''))
              const height=Number(height_str.replace('px',''))
              console.log(width, height)
              
              const svg = d3.select(mySVG.value)
                  .attr("width", width)
                  .attr("height", height);

              // Define scales
              const xScale = d3.scaleLinear()
              .domain([0, d3.max(data, d => d.x) as number])
              .range([0, width]);

              const yScale = d3.scaleLinear()
              .domain([-0.05, 1])
              .range([height, 0]);

              // Create a line generator
              const line = d3.line<{ x: number; y: number }>()
              .x(d => xScale(d.x))
              .y(d => yScale(d.y))
              .curve(d3.curveBumpX);              

              // Draw the line
              const path = svg.append('path')
              .datum(data)
              .attr('fill', 'none')
              .attr('stroke', 'steelblue')
              .attr('stroke-width', 1.5)
              .attr('d', line);
                            
              console.log('svg path appended:', path.node()?.getTotalLength())

              function pointColoring(ptid: number): string{                
                if (ptid == selectedPointID.value){
                  return 'red'
                }
                return 'steelblue'
              }

              // draw the points
              svg.selectAll("myCircles")
                .data(data)
                .enter()
                .append("circle")
                  .attr("fill", d => pointColoring(d.id))
                  .attr("stroke", "none")
                  .attr("cx", d => xScale(d.x))
                  .attr("cy", d => yScale(d.y))
                  .attr("r", 3)
              svg_context.value = {xScale, yScale , width, height, 'path': path.node()}
            }
            
        }

        function dist_vec2(pt1: [number, number], pt2: [number, number]){
          
          let val = Math.pow((pt1[0] - pt2[0]), 2)
          val += Math.pow((pt1[1] - pt2[1]), 2)
          return Math.sqrt(val)
        }

        function findNearestPoint(offsetX: number, offsetY: number){
          let neareast_id: number | null = null
          if (svg_context.value){
            const x = svg_context.value.xScale.invert(offsetX)
            const y = svg_context.value.yScale.invert(offsetY)
            let dist = 0.0
            const max_dist_cutoff = 1000
            let maxdist = max_dist_cutoff            
            for (let ipt = 0; ipt <= data.length - 1; ipt ++){
              dist = dist_vec2([x, y], [data[ipt].x, data[ipt].y])
              if (dist<maxdist && dist<=max_dist_cutoff){
                maxdist = dist 
                neareast_id = ipt 
              }
            }          
            
          }
          console.log('found nearest point', neareast_id)
          return neareast_id
        }

        function updatePlot(){
          // this just clears and rebuilds, would be better to update. 
          // figure that out later
          const svg = d3.select(mySVG.value)          
          svg.select('path').remove()
          svg.selectAll('circle').remove()          
          buildPlot()
        }

        onMounted(() => {
            console.log("we have mounted!")
            console.log(svgContainer.value)
            maxPointId.value =  Math.max(...data.map(d => d.id))
            buildPlot()            
        })
    
        function handleDoubleClick(event: MouseEvent){
            console.log('double clicked! adding new point', event)
            

            if (svg_context.value){
              
              let new_x = event.offsetX
              let new_y = event.offsetY
              new_x = svg_context.value.xScale.invert(new_x)
              new_y = svg_context.value.yScale.invert(new_y)
              if (new_y < 0){ 
                new_y = 0.0
              }

              const new_xy = {'x':new_x, 'y': new_y, 'id': maxPointId.value +1}              
              
              let new_id = 0;
              if (new_x < data[0].x){
                data.splice(0, 0, new_xy)
                new_id = 0
              } else if (new_x > data[data.length - 1].x){
                data.push(new_xy)
                new_id = data.length
              } else { 
                for (let id = 0; id< data.length-1; id++){
                   if (new_x > data[id].x && new_x < data[id+1].x){
                      data.splice(id+1, 0, new_xy)
                      new_id = id+1
                      break
                   }
                }
              }
              
              console.log("updated data at index", new_id, data[new_id])
              console.log(data)              
              selectedPoint.value = new_id // the array index
              selectedPointID.value = new_xy.id  // the unique identifier
              maxPointId.value = maxPointId.value  + 1           
              updatePlot()                            
            }
        }

        function mouseDown(event: MouseEvent){
            console.log(downClickTimes)
            downClickTimes[0] = downClickTimes[1]
            downClickTimes[1] = Date.now()            

            const elapsed_time = downClickTimes[1] - downClickTimes[0]
            
            if (elapsed_time < doubleClickTime_ms){
                mouseEventType.value = 'double-click'
                handleDoubleClick(event)
            } else { 
                mouseEventType.value = 'single-click-or-drag'                
                clickDownPosition[0] = event.offsetX
                clickDownPosition[1] = event.offsetY
                console.log("stored new position:", clickDownPosition)
            }
            console.log(event)
        }

        function mouseUp(event: MouseEvent){
            const upTime = Date.now()

            // get the time the mouse down was held
            const elapsed_time_ms = upTime - downClickTimes[1]

            if (mouseEventType.value !== 'double-click'){
                console.log("mouseup, not a double")
                let dx = event.offsetX - clickDownPosition[0]
                let dy = event.offsetY - clickDownPosition[1]

                if (dx === 0 && dy === 0 && elapsed_time_ms < 700){ 
                    // single click, no-drag
                    console.log('handle single click')

                    // find closest point within cutoff 
                    const ptid = findNearestPoint(event.offsetX, event.offsetY)
                    if (ptid !== null){
                      selectedPoint.value = ptid
                      selectedPointID.value = data[ptid].id
                    }
                    console.log('found closest point id:', selectedPoint.value)
                    updatePlot()
                } else if (selectedPoint.value !== null) { 
                    console.log("handling drag from ", clickDownPosition)
                    console.log("to ", [event.offsetX, event.offsetY])                    
                    
                    console.log('drag deltas: ', [dx, dy])

                    if (svg_context.value){
                      // scale the screen deltas to domain delta values
                      const xScale = svg_context.value.xScale
                      const yScale = svg_context.value.yScale

                      // x0 in screen range
                      const x0 = xScale(data[selectedPoint.value].x)
                      const y0 = yScale(data[selectedPoint.value].y)

                      // transform in screen space
                      const new_x = x0 + dx 
                      const new_y = y0 + dy
                      
                      
                      const new_x_domain = xScale.invert(new_x)
                      let new_y_domain = yScale.invert(new_y)
                      if (new_y_domain < 0){ 
                        new_y_domain = 0.0
                      }
                      
                      // update the data point                      
                      console.log(data[selectedPoint.value])
                      data[selectedPoint.value].x = new_x_domain
                      data[selectedPoint.value].y = new_y_domain 
                      console.log(data[selectedPoint.value])

                      // sort in case we crossed another point                      
                      data.sort((a, b) => a.x - b.x);
                      // and update the selected id in case ordering has changed
                      for (let ipt = 0; ipt <= data.length -1 ; ipt++){
                        if (data[ipt].id == selectedPointID.value){
                          selectedPoint.value = ipt 
                          break
                        }
                      }

                      // redraw the graph
                      updatePlot()
                    }                    
                }      
            }
        }


        
        return {mySVG, svgContainer, mouseDown, mouseUp, exportTF}
    }

}

</script>


<style scoped>
h1 {
  font-weight: 500;
  font-size: 2.6rem;
  position: relative;
  top: -10px;
}

h3 {
  font-size: 1.2rem;
}

.greetings h1,
.greetings h3 {
  text-align: center;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
