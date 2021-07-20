<template>
    <div class="center">
        <div id="art"></div>      
        <div>
            <span>
                <iv-button style="height: 40px;" @click="togglePlay">
                    {{ displayText }}
                </iv-button>
                <iv-button style="height: 40px; margin-left: 10px;" @click="reset=true">
                    Reset
                </iv-button>
            </span>
        </div>  
    </div>
</template>

<script>
import * as math from 'mathjs';
import P5 from 'p5';
import $ from 'jquery'; // eslint-disable-line

export default {
    name: "artillery",
    data() {
        return {
            image: require('@/assets/ocean.png'), // eslint-disable-line
            play: false,
            reset: false,
        };
    },
    props: {
        speed: Number,
        orientation: Number,
        latitude: Number,
    },
    computed: {
       displayText() {
           if (this.play) {
               return "Pause";
           } else {
               return "Play";
           }
       }
    },
    methods: {
        togglePlay() {
            this.play = !this.play;
        }
    },
    mounted() {
        /* eslint-disable no-unused-vars */
        let vm = this;
    
        let width = 450, height = 450;
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
                bacc = p5.loadImage(vm.image);
            }

            p5.setup = function () {
                let art = p5.createCanvas(width,height);
                art.parent("art")
            }

            let play = vm.play;
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
                if (vm.reset) {
                    vm.reset = false;
                    i = width/2;
                    j = height/2;
                    inc = i;
                    jnc = j;
                    counter = 0;
                    xpositions = [];
                    ypositions = [];
                    xpositionsnc = [];
                    ypositionsnc = [];
                }

                [[velx,vely,velz], [corx,cory], theta, epsilon, v] = computeValues();

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

                if (vm.play && ((i-width/2)**2+(j-height/2)**2)**0.5<(width*v/2)) {
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
                    vm.play = false;
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

        const p5inst = new P5(artillery, 'art');
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