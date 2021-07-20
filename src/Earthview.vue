<template>
    <div class="center">
        <div id="earth">
            
        </div>
        <span>
            <iv-button style="height: 40px;" @mouseover="toggleShow" @mouseleave="toggleShow">
                Earth Frame
            </iv-button>
            <iv-button style="height: 40px; margin-left: 10px;" @click="togglePlay">
                {{ buttonText }}
            </iv-button>
        </span>
    </div>
    
</template>

<script>
import P5 from 'p5';

export default {
    name: 'earthview',
    data() {
        return {
            image: require('@/assets/Earth_smaller.png'), // eslint-disable-line
            show: false,
            playing: true,
            buttonPressed: false,
        };
    },
    computed: {
        buttonText() {
            if (this.playing) {
                return "Pause";
            } else {
                return "Play";
            }
        }
    },
    methods: {
        toggleShow() {
            this.show = !this.show;
            console.log(`show: ${this.show}`);
        },
        togglePlay() {
            this.playing = !this.playing;
        },
    },
    mounted() {
        /* eslint-disable no-unused-vars */
        let vm = this;
        let wideness = 450, heightness = 450;
        // let wideness = $("#graph").width(), heightness = 0.7*$(window).height();
        let playing = this.playing;
        let omega;
        let buttonIncrease;
        let buttonDecrease;
        let show = this.show;
        let img;
        let dragging = true;
        const radius = wideness/3;
        let vecShow = false;
        let derShow = false;

        const earthview = p5 => {
            p5.preload = () => {
                img = p5.loadImage(this.image);
            }

            p5.setup = () => {
                let canvas = p5.createCanvas(wideness,heightness,"webgl");
                canvas.parent("earth");
            }

            p5.draw = () => {
                // omega = slider.value();
                if (this.playing) {
                    omega = 0.5;
                    let flatteningFactor = 1 - omega * omega;
                    let a = Math.sqrt(radius * radius / flatteningFactor);
                    p5.background(3);
                    p5.camera(2*wideness/3,0,2*wideness/3,0,0,0,0,1,0)

                    if (this.show) {
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
        }

        const p5inst = new P5(earthview, 'earth');

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
