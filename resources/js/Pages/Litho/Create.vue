<script setup>
import { ref, onMounted } from 'vue'

import { Head } from '@inertiajs/vue3';
import AuthenticatedLayout from "@/Layouts/AuthenticatedLayout.vue";

import * as THREE from 'three';
import {OrbitControls} from "three/addons/controls/OrbitControls";
import { STLExporter } from 'three/addons/exporters/STLExporter.js';
import {VertexNormalsHelper} from "three/addons/helpers/VertexNormalsHelper";

const exporter = new STLExporter();
const options = { binary: true }


const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 0.1, 1000 );
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });

onMounted(() => {
    scene.add(new THREE.AxesHelper(5))

    // Create ambient light and add to scene.
    var light = new THREE.AmbientLight(0xf7f7f7); // soft white light
    scene.add(light);

    // Create directional light and add to scene.
    var directionalLight = new THREE.DirectionalLight(0xffffff);
    directionalLight.position.set(1, 1, 1).normalize();
    scene.add(directionalLight);

    renderer.setSize( window.innerWidth, window.innerHeight );
    document.getElementById('main-container').appendChild(renderer.domElement );

    const boxGeometry = new THREE.BoxGeometry(80, 50, 3, 400, 400);

// Load the displacement map
    const displacementMap = new THREE.TextureLoader().load('http://localhost:8000/images/image.png', function (texture) {
        // Texture loaded callback
        displacementMap.image = texture.image; // Store the loaded image
        displacementMap.needsUpdate = true; // Update the texture

        function calculateUV(x, y, width, height)
        {
            return {
                u: ((x + (width / 2)) / width),
                v: 1 - ((y + (height / 2)) / height),
            };
        }

        // Create a function to displace the vertices based on the displacement map
        function displaceVertices(geometry, displacementMap, scale) {
            const vertices = geometry.attributes.position.array;
            const displacementImageData = getDisplacementImageData(displacementMap);

            if (!displacementImageData) {
                console.error('Failed to access displacement map image data.');
                return;
            }

            const imageWidth = displacementMap.image.width;
            const imageHeight = displacementMap.image.height;
            let uvCheck = [];
            let xyCheck = [];

            for (let i = 0; i < vertices.length; i += 3) {
                const x = vertices[i];
                const y = vertices[i + 1];

                let uvVals = calculateUV(x, y, geometry.parameters.width, geometry.parameters.height)

                // uvCheck.push(uvVals);
                // xyCheck.push({
                //     x: x,
                //     y: y
                // });

                // Sample the displacement map
                const value = getDisplacementValue(displacementImageData, uvVals.u, uvVals.v, imageWidth, imageHeight);

                //console.log(value);

                // Apply displacement to vertices
                vertices[i + 2] += value * scale; // Displace along the Z-axis
            }

            //console.log(uvCheck);
            //console.log(xyCheck);

            geometry.computeVertexNormals();
            geometry.attributes.position.needsUpdate = true;
        }

        // Helper function to get displacement image data
        function getDisplacementImageData(displacementMap) {
            const canvas = document.createElement('canvas');
            canvas.width = displacementMap.image.width;
            canvas.height = displacementMap.image.height;

            const context = canvas.getContext('2d');
            context.drawImage(displacementMap.image, 0, 0);
            console.log(context.getImageData(0, 0, displacementMap.image.width, displacementMap.image.height));
            try {
                return context.getImageData(0, 0, displacementMap.image.width, displacementMap.image.height);
            } catch (error) {
                console.error('Failed to get displacement image data:', error);
                return null;
            }
        }

        let indexs = []
        // Helper function to get displacement value from image data
        function getDisplacementValue(imageData, u, v, width, height) {
            const index = (Math.floor(u * width - 1) + (Math.floor(v * height - 1) * width)) * 4;
            const pixel = (imageData.data[index] + imageData.data[index + 1]) / 3; // Assuming it's grayscale

            indexs.push({
                u: u,
                v: v,
                index: index
            })

            return (pixel / 255) - 0.5; // Normalize to [-0.5, 0.5]
        }

        boxGeometry.computeVertexNormals();
        // Call the function to displace vertices
        displaceVertices(boxGeometry, displacementMap, 10); // Adjust the scale as needed

        // console.log(indexs);

        // Create a new mesh with the modified geometry
        const displacedMesh = new THREE.Mesh(boxGeometry, [
                new THREE.MeshLambertMaterial( {color: 'lightgray'}),
                new THREE.MeshLambertMaterial( {color: 'lightgray'}),
                new THREE.MeshLambertMaterial( {color: 'lightgray'}),
                new THREE.MeshLambertMaterial( {color: 'lightgray'}),
                new THREE.MeshPhongMaterial( {
                    map: texture,
                    // bumpMap: displacementMap,
                    // bumpScale: -2,
                    // displacementMap: displacementMap,
                    // displacementScale: -2,
                    // side: THREE.FrontSide
                }),
                new THREE.MeshLambertMaterial( {color: 'lightgray'}),
            ]);
        scene.add(displacedMesh);

        // Parse the input and generate the STL encoded output
        // const result = exporter.parse( plane, options );

        const link = document.createElement( 'a' );
        link.style.display = 'none';
        document.body.appendChild( link );

        function save( blob, filename ) {
            link.href = URL.createObjectURL( blob );
            link.download = filename;
            link.click();
        }

        function saveArrayBuffer( buffer, filename ) {
            save( new Blob( [ buffer ], { type: 'application/octet-stream' } ), filename );
        }

        //saveArrayBuffer( result, 'box.stl' );
    });

    // const plane = new THREE.Mesh(
    //     planeGeometry,
    //     [
    //         new THREE.MeshLambertMaterial( {color: 'lightgray'}),
    //         new THREE.MeshLambertMaterial( {color: 'lightgray'}),
    //         new THREE.MeshLambertMaterial( {color: 'lightgray'}),
    //         new THREE.MeshLambertMaterial( {color: 'lightgray'}),
    //         new THREE.MeshPhongMaterial( {
    //             map: texture,
    //             bumpMap: displacementMap,
    //             bumpScale: -2,
    //             displacementMap: displacementMap,
    //             displacementScale: -2,
    //             normalMap: normalMap,
    //             side: THREE.FrontSide
    //         }),
    //         new THREE.MeshLambertMaterial( {color: 'lightgray'}),
    //     ]
    // );
    // plane.matrixAutoUpdate = true;
    // scene.add(plane)

    window.addEventListener('resize', onWindowResize, false)
    function onWindowResize() {
        camera.aspect = window.innerWidth / window.innerHeight
        camera.updateProjectionMatrix()
        renderer.setSize(window.innerWidth, window.innerHeight)
        render()
    }
    camera.position.z = 3

    const controls = new OrbitControls(camera, renderer.domElement)
    controls.screenSpacePanning = true //so that panning up and down doesn't zoom in/out
    //controls.addEventListener('change', render)

});

function animate() {
    requestAnimationFrame(animate);

    render();
}

function render() {
    renderer.render(scene, camera);
}

animate();
</script>

<template>
    <Head title="Create" />

    <AuthenticatedLayout>
        <div id="options-container">

        </div>
        <div id="main-container">

        </div>
    </AuthenticatedLayout>
</template>

<style scoped>
    #options-container {

    }

    #main-container {
        background: #11e8bb; /* Old browsers */
        background: -moz-linear-gradient(top,  #11e8bb 0%, #8200c9 100%); /* FF3.6-15 */
        background: -webkit-linear-gradient(top,  #11e8bb 0%,#8200c9 100%); /* Chrome10-25,Safari5.1-6 */
        background: linear-gradient(to bottom,  #11e8bb 0%,#8200c9 100%); /* W3C, IE10+, FF16+, Chrome26+, Opera12+, Safari7+ */
        filter: progid:DXImageTransform.Microsoft.gradient( startColorstr='#11e8bb', endColorstr='#8200c9',GradientType=0 ); /* IE6-9 */
        overflow: hidden;
    }
</style>
