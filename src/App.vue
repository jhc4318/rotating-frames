<template> 
    <div>
        <iv-DraggableDiv idName="controlPanel" style="width: 20vw;">
            <div>
                <iv-slider id="speedSlider" name="Speed (relative)" :min="0" :max="1" :step="0.1" :init_val="0.5" :tick_step="1" @sliderChanged="changeSpeed" />
                <iv-slider id="orientationSlider" name="Velocity - Orientation" :min="0" :max="360" :step="22.5" :init_val="180" :tick_step="360" @sliderChanged="changeOrientation" />
                <iv-slider id="latitudeSlider" name="Latitude" :min="-90" :max="90" :step="11.25" :init_val="33.75" :tick_step="180" @sliderChanged="changeLatitude" />
                <iv-slider id="longitudeSlider" name="Longitude" :min="0" :max="360" :step="22.5" :init_val="45" :tick_step="360" @sliderChanged="changeLongitude" />
            </div>
        </iv-DraggableDiv>
        <iv-visualisation :title="projectName">
            <template #hotspots>
                <iv-pane position="left" format="full">
                    <iv-sidebar-content :showPagination="true">
                        <iv-sidebar-section title="Centrifugal" icon="globe">
                            The <span style="color: rgb(255,0,0)">red vector</span> on the "View Vectors" tab shows the direction of the centrifugal force. However, due to the redistribution of the Earth's mass making it an oblate spheroid, the centrifugal force does not affect the motion of objects in the Earth's frame.
                        </iv-sidebar-section>

                        <iv-sidebar-section title="Coriolis" theme="Lime" icon="undo">
                            We can express <iv-equation-box class="in-line-eqn" :stylise="false" equation="\boldsymbol{\omega}" /> in terms of the unit vectors shown in the diagram as <iv-equation-box class="in-line-eqn" :stylise="false" equation="\boldsymbol{\omega}=\omega(cos(\lambda)\mathbf{j}+sin(\lambda)\mathbf{k})" /> where <iv-equation-box class="in-line-eqn" :stylise="false" equation="\lambda" /> is the latitude. Recalling that the Coriolis force is given by <iv-equation-box class="in-line-eqn" :stylise="false" equation="\mathbf{F_{cor}}=-2m\mathbf{v}\times\boldsymbol{\omega}" />, and bearing in mind that the velocity only has components in the plane tangential to the Earth (only has <iv-equation-box class="in-line-eqn" :stylise="false" equation="\mathbf{i}" /> and <iv-equation-box class="in-line-eqn" :stylise="false" equation="\mathbf{j}" /> components), deviations in the plane of motion will only be contributed to by the <iv-equation-box class="in-line-eqn" :stylise="false" equation="\mathbf{k}" /> term in <iv-equation-box class="in-line-eqn" :stylise="false" equation="\boldsymbol{\omega}" />. <br><br>
                            Therefore, the magnitude of the coriolis acceleration in the plane of motion is given by <iv-equation-box class="in-line-eqn" :stylise="false" equation="a=2v\omega sin(\lambda)" />. Considering a projectile that travels with a constant speed in the plane tangential to the Earth for a distance <iv-equation-box class="in-line-eqn" :stylise="false" equation="L" /> in time <iv-equation-box class="in-line-eqn" :stylise="false" equation="t" /> (so <iv-equation-box class="in-line-eqn" :stylise="false" equation="t=\frac{L}{v}" />), the sideways deflection (in the Earth frame) due to the coriolis force is given by: <br><br>
                            <span class="center"><iv-equation-box equation="s = \frac{1}{2}(2\omega vsin(\lambda))(\frac{L}{v})^2 \\ = \frac{\omega L^2sin(\lambda)}{v}." /></span> <br>
                            Note: the coriolis force can be considered constant so long as the projectile does not travel far enough to change the lattitude.

                            <div class="center">
                                <img src="/assets/Diagram.PNG" alt="diagram of coriolis vector" style="height: 100%; width: 100%;" />
                            </div>
                        </iv-sidebar-section>

                        <iv-sidebar-section title="Instructions" theme="Gold" icon="book-open">
                            Open the "Parameters" hotspot to the right. Use the sliders to adjust various parameters and see how it changes the vectors and the path travelled on the right. <br><br>

                            Click the "View Earth" tab to see different view frames. <br>

                            <h4 style="margin-bottom: 10px">Colour Code:</h4>
                            <p style="border-left: solid 5px #FF00FF; padding-left: 15px; margin-bottom: 7px"> <b>Omega Vector</b></p>
                                <p style="border-left: solid 5px rgb(0,146,146); padding-left: 15px; margin-bottom: 7px"> <b>Position Vector</b></p>
                                <p style="border-left: solid 5px rgb(0,0,0); padding-left: 15px; margin-bottom: 7px"> <b>Velocity Vector</b></p>
                                <p style="border-left: solid 5px rgb(255,0,0); padding-left: 15px; margin-bottom: 7px"> <b>Centrifugal Force Vector</b></p>
                                <p style="border-left: solid 5px #008000; padding-left: 15px; margin-bottom: 15px"> <b>Coriolis Force Vector</b></p>
                        </iv-sidebar-section>
                    </iv-sidebar-content>
                </iv-pane>

                <!-- Toggles -->
                <iv-toggle-hotspot :draggable="true" position="bottom" title="Parameters" idName="controlPanel" />

                <iv-fixed-hotspot position="top" transparent>
                    <div class="center" style="z-index: 2; margin-right: 450px;">
                        <iv-toggle-advance :modes="['View Vectors', 'View Earth']" @toggleswitched="changeToggle" />
                    </div>   
                </iv-fixed-hotspot>
            </template>

            <div class="center">
                <div style="padding-top: 50px;">
                    <div class="graph-item">
                        <div id="graph"/>
                        <div v-if="currentToggle"><Earthview /></div>
                    </div>
                    <div class="graph-item">
                        <Artillery :speed="speed" :orientation="orientation" :latitude="latitude" />
                    </div>   
                </div>    
            </div>
            
            
            
        </iv-visualisation>
    </div>
</template>
<script>
import numeric from 'numeric';
import * as math from 'mathjs';
import Plotly from 'plotly.js-dist';
import Earthview from './Earthview.vue';
import Artillery from './Artillery.vue'

export default {
    name:"App",
    components: {
        Earthview, Artillery,
    },
    data(){
        return {
            projectName: "Pseudo-forces in Earth's rotating frame",
            speed: 0.5,
            orientation: 180,
            latitude: 33.75,
            longitude: 45,
            redrawPlot: false,
            reInitPlot: false,
            currentToggle: 0,
            // image: require('@/assets/Diagram.PNG'),
        };
    },
    methods: {
        changeSpeed(e) {
            this.speed = e;
            console.log(`speed: ${this.speed}`)
            this.redrawPlot = true;
        },
        changeOrientation(e) {
            this.orientation = e;
            console.log(`orientation: ${this.orientation}`)
            this.redrawPlot = true;
        },
        changeLatitude(e) {
            this.latitude = e;
            console.log(`latitude: ${this.latitude}`)
            this.redrawPlot = true;
        },
        changeLongitude(e) {
            this.longitude = e;
            console.log(`longitude: ${this.longitude}`)
            this.redrawPlot = true;
        },
        changeToggle(e) {
            this.currentToggle = e;
            if (this.currentToggle == 0) {
                this.reInitPlot = true;
            }
            this.redrawPlot = true;
        }
    },
    mounted() {
        /* eslint-disable no-unused-vars */
        let vm = this; 

        ///Colour definitions
        let magenta = "#FF00FF",
            blue = "#0000FF",
            green = "#008000",
            impBlue = "#003E74",
            red = "rgb(255,0,0)",
            black = "rgb(0,0,0)",
            white = "rgb(255,255,255)",
            cyan = "rgb(0,146,146)", //1
            lilac = "rgb(182,109,255)", //2
            orange = "rgb(219,209,0)"; //3

        // Plot utilities ------------------------------------------------
        function createView(point) {
        let norm = Math.sqrt(point[0]*point[0] + (5*point[1])*(5*point[1]) + point[2]*point[2]);
        let a = 0.5 + point[0]/norm, b = 1 +  5*point[1]/norm, c = 0.5 + point[2]/norm;
        let camera = {
            up: {x: 0, y: 0, z: 1},
            eye: {x: -a, y: -b, z: c + 0.5},
            center: {x: 0, y: 0, z: -0.2}
        }
        return camera
        }

        //Create Axes Object
        function createAxes(length) {
            let axes = [];
            let vector = [0, length];
            let initialVec;
            let vecString = ["x", "y", "z"];
            let originString = ["", "", "(0,0,0)"];
            for (let i = 0; i < 3; ++i) {
                initialVec = [[0, 0], [0, 0], [0, 0]];
                initialVec[i] = vector;
                axes.push(
                    {
                        type: "scatter3d",
                        mode: "lines+text",
                        x: initialVec[0],
                        y: initialVec[1],
                        z: initialVec[2],
                        line: {color: "rgb(0,0,0)", width: 4},
                        text: [originString[i], vecString[i]],
                        textfont: {size: 15},
                        textposition: "top"
                    }
                );
            }
            return axes;
        }

        ///Change of Basis:
        //Spherical 2 Cartesian
        function sp2c(r,theta,phi) {
            return [
                r * Math.sin(theta) * Math.cos(phi),
                r * Math.sin(theta) * Math.sin(phi),
                r * Math.cos(theta)
            ];
        }
        //Cartesian 2 Spherical
        function c2sp(x,y,z) {
            let r = 0, theta = 0, phi = 0;
            if (x*x + y*y + z*z !== 0) {
                r = Math.sqrt(x*x + y*y + z*z);
                theta = Math.acos(z/r);
                phi = Math.atan2(y,x);
            }
            return [r, theta, phi];
        }

        ///Shape Objects:
        //3D Objects:
        function Line(points) {
            this.x = [];
            this.y = [];
            this.z = [];
            for (let i = 0; i  < points.length; ++i) {
                this.x.push(points[i][0]);
                this.y.push(points[i][1]);
                this.z.push(points[i][2]);
            }

            this.gObject = function(color, width=7, dash="solid") {
                let lineObject = {
                    type: "scatter3d",
                    mode: "lines",
                    x: this.x,
                    y: this.y,
                    z: this.z,
                    line: {color: color, width: width, dash:dash}
                }
                return lineObject;
            }

            this.arrowHead = function(color, width=7, dash="solid") {
                let lastElm = this.x.length - 1;
                let [r, theta, phi] = c2sp(this.x[lastElm], this.y[lastElm], this.z[lastElm]);
                let frac = 0.1*r;
                let sin45 = Math.sin(Math.PI/4);
                let d = r - frac * sin45;
                let wingLength = Math.sqrt(Math.pow(frac*sin45,2) + d*d);
                let wingAngle = Math.acos(d/wingLength);


                let wings_xyz = [
                    sp2c(wingLength, theta + wingAngle, phi),
                    sp2c(wingLength, theta - wingAngle, phi)
                ];

                let wings = {
                    type: "scatter3d",
                    mode: "lines",
                    x: [wings_xyz[0][0], this.x[lastElm], wings_xyz[1][0]],
                    y: [wings_xyz[0][1], this.y[lastElm], wings_xyz[1][1]],
                    z: [wings_xyz[0][2], this.z[lastElm], wings_xyz[1][2]],
                    line: {color: color, width: width}
                }

                return wings;
            }
        }
        function Point(position) {
            this.position = position;
            this.gObject = function(color, size = 7, symbol="circle") {
                let pointObject = {
                    type: "scatter3d",
                    mode: "markers",
                    x: [this.position[0]],
                    y: [this.position[1]],
                    z: [this.position[2]],
                    marker: {"color": color, "size": size, "symbol": symbol}
                }
                return pointObject;
            }
        }
        function Sphere(radius) {
            this.radius = radius;
            let meshSize = 20;
            let theta = numeric.linspace(0, Math.PI, meshSize);
            let phi = numeric.linspace(0, 2*Math.PI, meshSize);
            this.x = [], this.y = [], this.z = [];
            for(let i = 0; i < meshSize; ++i) {
                this.x[i] = [], this.y[i] = [], this.z[i] = [];
                for(let j = 0; j < meshSize; ++j) {
                    this.x[i].push(radius*Math.cos(phi[i])*Math.sin(theta[j]));
                    this.y[i].push(radius*Math.sin(phi[i])*Math.sin(theta[j]));
                    this.z[i].push(radius*Math.cos(theta[j]));
                }
            }
            this.gObject = function(color1, color2) {
                let sphere = {
                    type: "surface",
                    x: this.x,
                    y: this.y,
                    z: this.z,
                    showscale: false,
                    opacity: 0.6,
                    colorscale: [[0.0, color1], [1.0, color2]]
                }
                return sphere;
            }
        }
        function Cylinder(radius, height){
            this.radius = radius;
            this.height = height;
            let meshSize = radius * 10;
            if (meshSize === 0) {meshSize = 2;}
            let phi = numeric.linspace(0, 2*Math.PI, meshSize);
            this.x = [], this.y = [], this.z = [];
            let hValue = numeric.linspace(-height, height, meshSize);

            let xTemp = [], yTemp = [], zTemp = [];
            for(let i = 0; i < meshSize; ++i) {
                xTemp.push(radius*Math.cos(phi[i]));
                yTemp.push(radius*Math.sin(phi[i]));
            }
            for(let i = 0; i < meshSize; ++i) {
                this.x.push(xTemp);
                this.y.push(yTemp);
                this.z.push(numeric.rep([meshSize], hValue[i]));
            }

            this.gObjectCurve = function(color1, color2) {
                let curve = {
                    type: "surface",
                    x: this.x,
                    y: this.y,
                    z: this.z,
                    showscale: false,
                    opacity: 0.7,
                    colorscale: [[0.0, color1], [1.0, color2]]
                }
                return curve;
            }
            this.gObjectTop = function(color) {
                let top = {
                    type: "scatter3d",
                    mode: "lines",
                    x: this.x[0],
                    y: this.y[0],
                    z: this.z[meshSize - 1],
                    line: {color: color.slice(0, -1) + ",0.2)", width: 1},
                    surfaceaxis: 2,
                    opacity: 0.5
                }
                return top;
            }
            this.gObjectBottom = function(color) {
                let bottom = {
                    type: "scatter3d",
                    mode: "lines",
                    x: this.x[0],
                    y: this.y[0],
                    z: this.z[0],
                    line: {color: color.slice(0, -1) + ",0.7)", width: 1},
                    surfaceaxis: 2,
                    opacity: 0.7
                }
                return bottom;
            }
        }
        function Cuboid(x, y, z){
            this.width = x,
            this.length = y,
            this.height = z,
            this.gObject = function(color) {
                let cuboid = {
                    type: "mesh3d",
                    x: [-x, -x, x, x, -x, -x, x, x],
                    y: [-y, y, y, -y, -y, y, y, -y],
                    z: [-z, -z, -z, -z, z, z, z, z],
                    i : [0, 0, 3, 4, 4, 4, 4, 4, 5, 6, 6, 7],
                    j : [2, 3, 4, 3, 6, 7, 1, 5, 2, 2, 7, 3],
                    k : [1, 2, 0, 7, 5, 6, 0, 1, 1, 5, 2, 2],
                    opacity: 0.5,
                    colorscale: [['0', color], ['1', "rgb(255,255,255)"]],
                    intensity: [0, 0.1, 0.3, 0.5, 0.7, 0.8, 0.9, 1],
                    showscale: false
                }
                return cuboid
            }
        }
        //2D Objects:
        function Circle(radius) {
            this.radius = radius;

            this.gObject = function(color=cyan, centre=[0,0]) {
                let phi = numeric.linspace(0, 2*Math.PI, 16);
                let x = [], y = [];
                for (let i=0, n=phi.length; i<n; ++i) {
                    x.push(this.radius*Math.cos(phi[i]) + centre[0]);
                    y.push(this.radius*Math.sin(phi[i]) + centre[1]);
                }
                let circle = {
                    type: "scatter",
                    mode: "lines",
                    x: x,
                    y: y,
                    line: {simplify: false, color: color},
                    fill:'tonexty',
                }
            }
        }

        // Global Initial Parameters --------------------------------------
        let initialPoint = [2., 2., 2.];
        let layout = {
            width: 450, height: 450,
            autosize: true,
            margin: {l:0, r:0, t:0, b:0},
            hovermode: "closest",
            showlegend: false,
            scene: {
                camera: createView([1,1,1]),
                xaxis: {range: [-10, 10], zeroline: true, autorange: false,},
                yaxis: {range: [-10, 10], zeroline: true, autorange: false,},
                zaxis: {range: [-10, 10], zeroline: true, autorange: false,},
                aspectratio: {x:1, y:1, z:1},
            }
        };
        let currentPoint = initialPoint;
        let initialPhi = 0, initialR = 6.37, initialTheta = 0, initialV = 0.5, initialEpsilon = 1;

        //Physics
        function findValues() {
            console.log("findValues");
            let r = 6.37;
            let omega = 2*Math.PI/8.6400;
            let v = vm.speed;
            let theta = vm.latitude * Math.PI / 180;
            let phi = vm.longitude * Math.PI / 180;
            let epsilon = vm.orientation * Math.PI / 180;


            let R = [r*Math.cos(phi)*Math.cos(theta), r*Math.sin(phi)*Math.cos(theta), r*Math.sin(theta)];
            let Omegamatrix = [0, 0, omega];
            let B = math.multiply(math.cross(Omegamatrix, R),-1);
            let Fcent = math.cross(Omegamatrix, B);

            let V = [v*(Math.cos(epsilon)*Math.sin(theta)*Math.cos(phi)-Math.sin(epsilon)*Math.sin(phi)),v*(Math.cos(epsilon)*Math.sin(theta)*Math.sin(phi)+Math.sin(epsilon)*Math.cos(phi)),-v*Math.cos(epsilon)*Math.cos(theta)];
            /*let V = [2*v*Math.cos(phi)*Math.sin(theta), 2*v*Math.sin(phi)*Math.sin(theta),(-2)*v*Math.cos(theta)];*/
            let vHat = math.multiply(V,-2);
            let Fcor = math.cross(Omegamatrix, vHat);
            return [V, Fcent, Fcor];
        }

        //Plots
        function initPlot() {
            console.log("initPlot");
            Plotly.purge("graph");

            //initialR = Math.sqrt(currentPoint[0]*currentPoint[0] + currentPoint[1]*currentPoint[1] + currentPoint[2]*currentPoint[2]);
            initialTheta = Math.atan2(currentPoint[2], Math.sqrt(currentPoint[0]*currentPoint[0] + currentPoint[1]*currentPoint[1]))/Math.PI;
            initialPhi = ((Math.atan2(currentPoint[1], currentPoint[0]) + 2*Math.PI) % (2*Math.PI))/Math.PI;
            //initialR = Math.round(initialR*10)/10;
            initialTheta = Math.round(initialTheta*16)/16;
            initialPhi = Math.round(initialPhi*8)/8;
            initialV = 0.5;
            initialEpsilon = Math.round(1*8/8);

            let v = vm.speed;
            let r = 6.37;
            let omega = 2*Math.PI/86400;
            let theta = vm.latitude * Math.PI / 180;
            let phi = vm.longitude * Math.PI / 180;
            let epsilon = vm.orientation * Math.PI / 180;

            Plotly.newPlot("graph", computeSpherical(r, theta, phi), layout);
            return;
        }
        
        //Plot for basis for corresponding coordinate systems:
        function computeSpherical(r, theta, phi, v) {
            console.log("computeSpherical");
            let sphere = new Sphere(r);
            let x = r*Math.cos(phi)*Math.cos(theta);
            let y = r*Math.sin(phi)*Math.cos(theta);
            let z = r*Math.sin(theta);
            let [[Vx, Vy, Vz], [Fcentx, Fcenty, Fcentz], [Fcorx, Fcory, Fcorz]] = findValues();


            currentPoint = [Math.round(x*10)/10, Math.round(y*10)/10, Math.round(z*10)/10];
            let data = [
                sphere.gObject("rgb(0,77,178)", white),
                //omega vector
                new Line([[0, 0, 0],[0, 0, r+1]]).gObject(magenta),
                new Line([[0, 0, r],[0, 0, r+1]]).arrowHead(magenta),

                //new Line([[x, y, z], [x + Math.cos(phi)*Math.cos(theta), y + Math.sin(phi)*Math.cos(theta), z + Math.sin(theta)]]).gObject(cyan),

                // r vector
                new Line([[0, 0, 0], [x, y, z]]).gObject(cyan),
                //new Line([[x, y, z], [x - Math.sin(phi), y + Math.cos(phi), z]]).gObject(lilac),

                //velocity vector
                new Line([[x, y, z], [x + 2*Vx, y + 2*Vy, z + 2*Vz]]).gObject(black),
                //new Line([[x, y, z], [x + Math.cos(theta)*Math.cos(phi), y + Math.cos(theta)*Math.sin(phi), z]]).gObject(red),

                //centrifugal and coriolis force vectors
                new Line([[x, y, z], [x + 0.5*Fcentx, y + 0.5*Fcenty, z + 0.5*Fcentz]]).gObject(red),
                new Line([[x, y, z], [x + 2*Fcorx, y + 2*Fcory, z + 2*Fcorz]]).gObject(green),
            ];

            let meshSize, t;
            let xTrace = [], yTrace = [], zTrace = [];

            // console.log(data);
            return data;
        }

        function updatePlot() {
            console.log("updatePlot");
            let data = [];

            if (vm.currentToggle == 0) {
                data = computeSpherical(
                    6.37,
                    vm.latitude * Math.PI / 180,
                    vm.longitude * Math.PI / 180,
                );

                Plotly.react( // FIX: This should be animate, but wasn't working and not sure why
                    'graph',
                    {data: data, layout: layout},
                    {
                        fromcurrent: true,
                        transition: {duration: 0,},
                        frame: {duration: 0, redraw: false,},
                        mode: "afterall",
                    }
                );
            } else {
                Plotly.purge("graph");
            }
        }

        function main() {
            console.log("main");
            initPlot();
        }
        
        function redraw() {
            requestAnimationFrame(redraw);

            if (vm.reInitPlot) {
                initPlot();
                vm.reInitPlot = false;
            }

            if (vm.redrawPlot) {
                updatePlot();
                vm.redrawPlot = false;
            }
        }

        main();
        redraw();
    },
}
</script>
<style>
.center{
    display:flex;
    flex-direction: column;
    align-items: center;
    /* margin-top: 50px; */
}
.in-line-eqn {
    margin-top: -25px;
    margin-bottom: -25px;
}
.graph-item {
    display: inline-block; 
    margin-top: 50px; 
    margin-left: 20px; 
    margin-right: 20px;
}
.iv-main-stage {
    overflow: hidden;
    overflow-x: hidden;
}
</style>