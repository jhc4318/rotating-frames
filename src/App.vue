<template> 
    <div>
        <iv-visualisation :title="projectName">
            <template #hotspots>     
                <!-- Sliders -->
                <iv-fixed-hotspot position="topright" style="z-index: 2;" transparent>
                    <div>
                        <iv-slider id="speedSlider" name="Speed (relative)" :min="0" :max="1" :step="0.1" :init_val="0.5" :tick_step="1" @sliderChanged="changeSpeed" />
                        <iv-slider id="orientationSlider" name="Velocity - Orientation" :min="0" :max="360" :step="22.5" :init_val="180" :tick_step="360" @sliderChanged="changeOrientation" />
                        <iv-slider id="latitudeSlider" name="Latitude" :min="-90" :max="90" :step="11.25" :init_val="33.75" :tick_step="180" @sliderChanged="changeLatitude" />
                        <iv-slider id="longitudeSlider" name="Longitude" :min="0" :max="360" :step="22.5" :init_val="45" :tick_step="360" @sliderChanged="changeLongitude" />
                    </div>
                </iv-fixed-hotspot>

                <!-- Toggles -->
                <iv-fixed-hotspot position="top" transparent>
                    <div class="center" style="z-index: 2;">
                        <iv-toggle-advance :modes="['View Earth', 'View Vectors']" @toggleswitched="changeToggle" />
                    </div>   
                </iv-fixed-hotspot>
            </template>

            <div v-if="!currentToggle" id="graph" class="center" style="padding-top: 50px;" />
            <div v-if="currentToggle" class="center">
                <Earthview />
            </div>
            
        </iv-visualisation>
    </div>
</template>
<script>
import {name} from '../package.json';
import $ from 'jquery'; // eslint-disable-line
import numeric from 'numeric';
import * as math from 'mathjs';
import Plotly from 'plotly.js-dist';
import P5 from 'p5'; // eslint-disable-line
import Earthview from './Earthview.vue';

export default {
    name:"App",
    components: {
        Earthview,
    },
    data(){
        return {
            projectName: name,
            speed: 0.5,
            orientation: 180,
            latitude: 33.75,
            longitude: 45,
            redrawPlot: false,
            reInitPlot: false,
            currentToggle: 0,
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
            width: 450, height: 500,
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

            console.log(data);
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

                Plotly.react(
                    'graph',
                    {data: data},
                    {
                        fromcurrent: true,
                        transition: {duration: 0,},
                        frame: {duration: 0, redraw: false,},
                        mode: "afterall",
                        showlegend: false,
                    }
                );
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


        //This file deals with the rotating Earth vis.
        /*
        let wideness = $("#graph").width(), heightness = 0.7*$(window).height();
        let playing = true;
        let omega;
        let buttonIncrease;
        let buttonDecrease;
        let show = false;
        let img;
        let dragging = true;
        const radius = wideness/3;
        let vecShow = false;
        let derShow = false;
        // $("input[type=range]").on('click', () => {dragging != dragging});
        // $("input[type=range]").on('mouseup', () => {dragging = false});

        const earthview = p5 => {
            p5.preload = () => {
                img = p5.loadImage("assets/Earth_smaller.png");
            }

            p5.setup = () => {
                let canvas = p5.createCanvas(wideness,heightness,"webgl");
                canvas.parent("earth");

                let buttonFrame = p5.createButton("Earth Frame");
                let buttonPlaying = p5.createButton('Play/Pause');
                buttonPlaying.parent("earth");
                buttonFrame.parent("earth");

                buttonFrame.mouseOver(function() {
                    show = !show;
                })

                buttonFrame.mouseOut(function() {
                    show = !show;
                })

                buttonPlaying.mousePressed(function() {
                    if (playing) {
                    p5.noLoop()
                    playing = false
                } else {
                    p5.loop()
                    playing = true
                }
                });
            }

            p5.draw = () => {
                // omega = slider.value();
                omega = 0.5;
                let flatteningFactor = 1 - omega * omega;
                let a = Math.sqrt(radius * radius / flatteningFactor);
                p5.background(3);
                p5.camera(2*wideness/3,0,2*wideness/3,0,0,0,0,1,0)


                if (show) {
                    p5.camera(Math.sqrt(2)*2*wideness/3*Math.sin((3*omega)*p5.millis()/1000+Math.PI/3), 0, Math.sqrt(2)*2/3*wideness*Math.cos((3*omega)*p5.millis()/1000+Math.PI/3), 0, 0, 0, 0, 1, 0);
                } // There was an empty else here

                p5.rotateY((3*omega)*p5.millis()/1000);
                p5.pointLight(250,250,250,0, 20)
                p5.pointLight(250,250,250,0, 20)
                p5.pointLight(250,250,250,0, 20)
                p5.pointLight(250,250,250,0, 20)
                p5.pointLight(250,250,250,0, 20)

                p5.texture(img);
                p5.sphere(radius)
            }
        }

        

        $(document).ready(function () {
            $("#graph").hide();
            $("#derivation").hide();

            $("#showVect").on('click', () => {
                vecShow = !vecShow;
                //document.getElementById("#showVect").value = (vecShow) ? "Hide Vectors" : "Show Vectors";
                if (vecShow) {
                    $("#earth").hide();
                    $("#graph").show();
                } else {
                    $("#earth").show();
                    $("#graph").hide();
                }
            });

            $("#showDer").on('click', () => {
                derShow = !derShow;
                //document.getElementById("showDer").value = (derShow) ? "Derivation/Explanation" : "Graphics/Sliders";
                if (derShow) {
                    $("#graphics").hide();
                    $("#derivation").show();
                } else {
                    $("#graphics").show();
                    $("#derivation").hide();
                }
            });
        });

        
        //This file deals with the 2-d motion vis. It also calls the functions defined in this file and earthView.js, initialising the p5 windows.
        let width = $("#art").width(), height = 0.7*$(window).height();
        const m = 50;

        // these arrays will log the positions of the particle for each frame.
        let counter = 0,
            xpositions = [],
            ypositions = [],
            xpositionsnc = [],
            ypositionsnc = [];
        let velx,
            vely,
            velz,
            corx,
            cory,
            s;

        let i = width/2,
            j = height/2,
            inc = i,
            jnc = j;

        //physics
        function computeValues() {
            let v = vm.speed;
            let epsilon = vm.orientation * Math.PI / 180;
            let theta = vm.latitude * Math.PI / 180;

            let vPrime = math.multiply([Math.cos(epsilon), Math.sin(epsilon), 0],-2*v);
            let omegaPrime = [0, 2*Math.PI/8.6400*Math.cos(theta), 2*Math.PI/8.6400*Math.sin(theta)];

            let FcorPrime = math.cross(vPrime,omegaPrime);
            let FcorPlane = [FcorPrime[0],FcorPrime[1]];

            return [vPrime, FcorPlane, theta, epsilon, v];
        }

        let theta, epsilon, v;
        [[velx,vely,velz], [corx,cory], theta, epsilon, v] = computeValues();

        //Ensures that the values will be updated whenever there is a change in the sliders.
        $(document).ready(function () {
            $("input[type=range]").each(function () {
                $(this).on("input", function () {
                    [[velx,vely,velz], [corx,cory], theta, epsilon, v] = computeValues();
                });
            });
        });
        let bacc;
        //p5

        const artillery = p5 => {
            p5.preload = () => {
                bacc = p5.loadImage("assets/ocean.png");
            }

            p5.setup = function () {
                let art = p5.createCanvas(width,height);
                art.parent("art")
            }

            let play = false;
            $(".play").on("click", () => {
                play = !play;
            });

            $(".reset").on("click", () =>{
                i = width/2;
                j = height/2;
                inc = i;
                jnc = j;
                counter = 0;
                xpositions = [];
                ypositions = [];
                xpositionsnc = [];
                ypositionsnc = [];
            })

            p5.draw = () => {
                p5.background(bacc);
                p5.rectMode(p5.rect.CENTER);
                for (let w=0;w<height;++w) {
                    if (w%80 === 0) {
                        //p5.stroke(19, 109, 0)
                        p5.stroke(255, 223, 145)
                        p5.line(w-40,height,w-40,0)
                        p5.line(0,w-40,width,w-40)
                    }
                }
                p5.noStroke();
                p5.fill(206, 0, 161);
                p5.ellipse(i, j,8,8);

                if (play && ((i-width/2)**2+(j-height/2)**2)**0.5<(width*v/2)) {
                    i -= (vely + counter*cory/(10*m));
                    j -= (velx + counter*corx/(10*m));
                    xpositions.push(i);
                    ypositions.push(j);

                    inc -= vely;
                    jnc -= velx;
                    xpositionsnc.push(inc);
                    ypositionsnc.push(jnc);

                    counter += 1;
                } else {
                    play = false;
                }

                for (let t=1; t<xpositions.length; t++) {
                    if (t%15 === 0) {
                        p5.stroke(255, 223, 145);
                        p5.line(xpositionsnc[t-6],ypositionsnc[t-6],xpositionsnc[t],ypositionsnc[t]);
                        p5.line(xpositionsnc[t-6]+1,ypositionsnc[t-6],xpositionsnc[t]+1,ypositionsnc[t]);
                        p5.stroke(255, 223, 145);
                        p5.line(xpositions[t-6],ypositions[t-6],xpositions[t],ypositions[t]);
                        p5.line(xpositions[t-6]+1,ypositions[t-6],xpositions[t]+1,ypositions[t]);

                        p5.line(xpositionsnc[t-6]+1,ypositionsnc[t-6],xpositionsnc[t]+1,ypositionsnc[t]);
                    }
                }

                p5.stroke(27, 173, 151);
                p5.fill(80, 88, 37);
                //p5.rotate(2);
                p5.ellipse(width/2,height/2,24,24);


                s = 2*Math.PI*(xpositions.length*100)**2*Math.sin(theta)/(86400*v*100)

                p5.fill(200,200,210);
                p5.noStroke();
                p5.textSize(16);
                p5.textStyle(p5.BOLD);

                p5.text("Range:    "+(width*v/2*0.1).toFixed(2)+"km",1.8*width/3,3.5*height/4);
                p5.text("Distance:  "+(Math.round(((i-width/2)**2+(j-height/2)**2)**0.5)*0.1).toFixed(2)+"km",1.8*width/3,3.7*height/4);
                p5.text("Deflection: "+(Math.round(s)).toFixed(2)+"m",1.8*width/3,3.9*height/4);

            };
        }

        const p5inst = new P5(earthview, 'earth');
        // const p5inst2 = new P5(artillery)
        */
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
</style>