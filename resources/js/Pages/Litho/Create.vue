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
    const displacementMap = new THREE.TextureLoader().load('http://127.0.0.1:8000/images/skra-bw.jpg', function (texture) {
        // Texture loaded callback
        displacementMap.image = texture.image; // Store the loaded image
        displacementMap.needsUpdate = true; // Update the texture

        // Create a function to displace the vertices based on the displacement map
        function displaceVertices(geometry, displacementMap, scale) {
            const vertices = geometry.attributes.position.array;
            const displacementImageData = getDisplacementImageData(displacementMap);

            if (!displacementImageData) {
                console.error('Failed to access displacement map image data.');
                return;
            }

            const width = displacementMap.image.width;
            const height = displacementMap.image.height;

            for (let i = 0; i < vertices.length; i += 3) {
                const x = vertices[i];
                const y = vertices[i + 1];
                const u = ((x / 80 + 0.5) * width) % width;
                const v = ((y / 50 + 0.5) * height) % height;

                // Sample the displacement map
                const value = getDisplacementValue(displacementImageData, u, v, width);

                //console.log(value);

                // Apply displacement to vertices
                vertices[i + 2] += value * scale; // Displace along the Z-axis
            }

            geometry.computeVertexNormals();
            geometry.attributes.position.needsUpdate = true;
        }

        // Helper function to get displacement image data
        function getDisplacementImageData(displacementMap) {
            const canvas = document.createElement('canvas');
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

        // Helper function to get displacement value from image data
        function getDisplacementValue(imageData, u, v, width) {
            const index = Math.floor(u) + Math.floor(v) * width;
            const pixel = imageData.data[index * 4]; // Assuming it's grayscale

            return (pixel / 255 - 0.5); // Normalize to [-0.5, 0.5]
        }

        boxGeometry.computeVertexNormals();
        // Call the function to displace vertices
        displaceVertices(boxGeometry, displacementMap, .6); // Adjust the scale as needed

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
