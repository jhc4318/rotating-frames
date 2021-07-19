<template>
    <div id="earth">
    </div>
</template>

<script>
import P5 from 'p5';

export default {
    name: 'earthview',
    data() {
        return {
            image: require('@/assets/Earth_smaller.png') // eslint-disable-line
        };
    },
    mounted() {
        /* eslint-disable no-unused-vars */
        let wideness = 400, heightness = 400;
        // let wideness = $("#graph").width(), heightness = 0.7*$(window).height();
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

        const earthview = p5 => {
            p5.preload = () => {
                img = p5.loadImage(this.image);
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

        const p5inst = new P5(earthview, 'earth');
    },
}

</script>

<style>

</style>
