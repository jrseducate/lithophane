<!--suppress PointlessArithmeticExpressionJS -->
<script setup>
import {ref, onMounted} from 'vue'

import {Head} from '@inertiajs/vue3';
import AuthenticatedLayout from "@/Layouts/AuthenticatedLayout.vue";

import * as THREE from 'three';
import {OrbitControls} from "three/addons/controls/OrbitControls";
import {STLExporter} from 'three/addons/exporters/STLExporter.js';
import {GLTFLoader} from 'three/addons/loaders/GLTFLoader.js';
import {VertexNormalsHelper} from "three/addons/helpers/VertexNormalsHelper";

const exporter = new STLExporter();
const options = {binary: true}

function createElement(options) {
    const tag = options.tag;
    delete options.tag;

    const element = document.createElement(tag);
    const keys = Object.keys(options);

    for (let i = 0; i < keys.length; i++) {
        const key = keys[i];
        const value = options[key];

        switch (key) {
            case 'children':
                const children = value;

                for (let j = 0; j < children.length; j++) {
                    element.appendChild(createElement(children[j]));
                }
                break;

            case 'classList':
                const classList = value;

                for (let j = 0; j < classList.length; j++) {
                    element.classList.add(classList[j]);
                }
                break;

            case 'style':
                const style = value;
                const styleKeys = Object.keys(style);

                for (let j = 0; j < styleKeys.length; j++) {
                    const styleKey = styleKeys[j];
                    const styleValue = style[styleKey];

                    element.style[styleKey] = styleValue;
                }
                break;

            default:
                element[key] = value;
                break;
        }
    }

    return element;
}

class ModelViewer {
    constructor() {
    }

    optionsAddImageUpload(name, onUpload) {
        this.options.appendChild(createElement({
            tag: 'div',
            style: {
                width: '100%',
            },
            children: [
                {
                    tag: 'label',
                    innerText: name,
                },
                {
                    tag: 'input',
                    type: 'file',
                    onchange: onUpload,
                }
            ]
        }));
    }

    optionsAddToggle(name, value, onUpdate) {
        this.options.appendChild(createElement({
            tag: 'div',
            style: {
                width: '100%',
            },
            children: [
                {
                    tag: 'label',
                    innerText: name,
                },
                {
                    tag: 'div',
                    children: [
                        {
                            tag: 'label',
                            className: 'switch',
                            children: [
                                {
                                    tag: 'input',
                                    type: 'checkbox',
                                    checked: value,
                                    onchange: onUpdate,
                                },
                                {
                                    tag: 'span',
                                    className: 'switch-slider round',
                                }
                            ]
                        }
                    ]
                }
            ]
        }));
    }

    optionsAddSlider(name, value, min, max, onUpdate) {
        this.options.appendChild(createElement({
            tag: 'div',
            style: {
                width: '100%',
            },
            children: [
                {
                    tag: 'label',
                    innerText: name,
                },
                {
                    tag: 'div',
                    style: {
                        display: 'flex',
                    },
                    children: [
                        {
                            tag: 'span',
                            style: {
                                marginRight: '15px',
                            },
                            innerText: min,
                        },
                        {
                            tag: 'input',
                            className: 'slider',
                            style: {
                                marginTop: '4px',
                            },
                            type: 'range',
                            min: min,
                            max: max,
                            value: value,
                            oninput: onUpdate,
                        },
                        {
                            tag: 'span',
                            style: {
                                marginLeft: '15px',
                            },
                            innerText: max,
                        },
                    ]
                }
            ]
        }));
    }

    getAspectRatio() {
        return this.renderer.domElement.clientWidth / this.renderer.domElement.clientHeight;
    }

    delay(delay, name, onTrigger) {
        const timeoutKey = 'timeout' + name;

        if (this[timeoutKey]) {
            clearTimeout(this[timeoutKey]);

            this[timeoutKey] = null;
        }

        this[timeoutKey] = setTimeout(onTrigger, delay);
    }

    loadImage(e) {
        var reader = new FileReader();
        reader.onload = (event) => {
            this.onImageSelected(event.target.result);
        }
        reader.readAsDataURL(e.target.files[0]);
    }

    onMount() {
        this.container = document.getElementById('main-container');
        this.options = document.getElementById('options-container');

        this.scale = 10;
        this.depth = 1;
        this.curvature = 0;
        this.greyscale = false;

        this.init();
    }

    onImageSelected(imageURL) {
        this.imageURL = imageURL;

        this.meshInit(() => {
            this.optionsInit();
        });
    }

    init() {
        this.sceneInit();
        this.rendererInit();
        this.cameraInit(80);

        this.optionsInit();
        this.bindingsInit();

        this.animate();
    }

    sceneInit() {
        this.scene = new THREE.Scene();

        this.sceneAddAxisHelper();
        this.sceneAddLight();
        this.sceneAddDirectionalLight();
    }

    sceneAddAxisHelper() {
        this.scene.add(new THREE.AxesHelper(5))
    }

    // Create ambient light and add to scene.
    sceneAddLight() {
        var light = new THREE.AmbientLight(0xf7f7f7); // soft white light

        this.scene.add(light);
    }

    // Create directional light and add to scene.
    sceneAddDirectionalLight() {
        var directionalLight = new THREE.DirectionalLight(0xffffff);

        directionalLight.position.set(1, 1, -100).normalize();

        this.scene.add(directionalLight);
    }

    rendererInit() {
        this.renderer = new THREE.WebGLRenderer({antialias: true, alpha: true});

        this.renderer.setSize(this.container.clientWidth, this.container.clientHeight);

        this.container.appendChild(this.renderer.domElement);
    }

    cameraInit(distanceZ) {
        this.camera = new THREE.PerspectiveCamera(75, this.getAspectRatio(), 0.1, 1000);

        this.camera.position.z = distanceZ;

        this.controls = new OrbitControls(this.camera, this.renderer.domElement)
        this.controls.screenSpacePanning = true //so that panning up and down doesn't zoom in/out
    }

    geometryInit(width, height, onLoadGeometry) {
        let aspectRatio = width / height;

        //this.geometry = new THREE.BoxGeometry(this.scale * aspectRatio, this.scale, 3, 400, 400);
        //console.log(this.geometry);

        const loader = new GLTFLoader()
        loader.load(
            'http://127.0.0.1:8000/models/box3.gltf',
            (meshScene) => {
                this.geometry = meshScene.scene.children[0].geometry;
                this.geometry.computeVertexNormals();
                this.displaceVertices(this.texture.image);
                onLoadGeometry.call(this);
            },
            (xhr) => {
                console.log((xhr.loaded / xhr.total) * 100 + '% loaded')
            },
            (error) => {
                console.log(error)
            }
        )
    }

    textureInit(imageURL, onTextureLoaded) {
        new THREE.TextureLoader().load(imageURL, onTextureLoaded.bind(this));
    }

    meshReload() {
        this.meshInit();
    }

    optionsInit() {
        this.options.innerHTML = '';

        this.optionsAddImageUpload('Texture', (e) => this.delay(0, 'set_texture', () => {
            this.loadImage(e);
        }));

        if (this.texture) {
            this.optionsAddSlider('Scale', this.scale, 10, 500, (e) => this.delay(0, 'set_scale', () => {
                this.scale = e.target.value;

                this.meshReload();
            }));

            this.optionsAddToggle('Greyscale', this.greyscale, (e) => this.delay(0, 'set_greyscale', () => {
                this.greyscale = e.target.checked;

                this.meshReload();
            }));

            this.optionsAddSlider('Depth', this.depth, 1, 100, (e) => this.delay(0, 'set_depth', () => {
                this.depth = e.target.value;

                this.meshReload();
            }));

            this.optionsAddSlider('Curvature', this.curvature * 100, 0, 100, (e) => this.delay(0, 'set_curvature', () => {
                this.curvature = e.target.value / 100;

                this.meshReload();
            }));
        }
    }

    bindingsInit() {
        window.addEventListener('resize', this.onResize.bind(this), false);
    }

    calculateUV(x, y, width, height) {
        let Two = 1.9999;

        return {
            u: ((x + (width / Two)) / width),
            v: 1.0001 - ((y + (height / Two)) / height),
        };
    }

    displaceVertices(image) {
        const geometry = this.geometry;
        const depth = this.depth;
        const curvature = this.curvature;
        let radius = 120;

        radius /= curvature * Math.PI;

        geometry.computeVertexNormals();

        const context = this.getImageCanvas(image);
        const imageData = context.getImageData(0, 0, image.width, image.height);

        if (!imageData) {
            return console.error('Failed to access image data.');
        }

        const imageWidth = image.width;
        const imageHeight = image.height;

        const yAxis = new THREE.Vector3(0, 1, 0);
        const vertexDatas = [];

        const vertices  = geometry.attributes.position.array;
        const normals   = geometry.attributes.normal.array;
        const uvs       = geometry.attributes.uv.array;
        const displace  = geometry.attributes._displace.array;

        const geometryWidth  = (geometry.boundingBox.max.x - geometry.boundingBox.min.x) * this.scale;
        const geometryHeight = (geometry.boundingBox.max.y - geometry.boundingBox.min.y) * this.scale;

        for (let i = 0; i < vertices.length; i += 3) {
            let vertex = new THREE.Vector3(
                vertices[i + 0],
                vertices[i + 1],
                vertices[i + 2]
            );

            console.log(geometryWidth, geometryHeight)

            const uv    = this.calculateUV(vertex.x - geometry.boundingBox.min.x, vertex.y - geometry.boundingBox.min.y, geometryWidth, geometryHeight);
            const value = this.getDisplacementValue(imageData, uv.u, uv.v, imageWidth, imageHeight);

            vertexDatas[i] = {
                uv: uv,
                value: value,
            };

            if (curvature > 0) {
                const center = new THREE.Vector3(
                    0,
                    vertex.y,
                    0
                );

                const direction = (new THREE.Vector3(0, 0, radius)).applyAxisAngle(yAxis, THREE.MathUtils.degToRad((uv.u - 0.5) * 360 * curvature));//.subVectors(vertex, center).normalize().multiplyScalar(radius);
                vertex = (new THREE.Vector3()).addVectors(center, direction).sub(new THREE.Vector3(0, 0, radius));

                vertices[i + 0] = vertex.x;
                vertices[i + 1] = vertex.y;
                vertices[i + 2] = vertex.z;
            }
        }

        geometry.computeVertexNormals();

        for (let i = 0; i < vertices.length; i += 3) {
            let vertexData = vertexDatas[i];
            let vertex = new THREE.Vector3(
                vertices[i + 0],
                vertices[i + 1],
                vertices[i + 2]
            );
            const normal = new THREE.Vector3(
                normals[i + 0],
                normals[i + 1],
                normals[i + 2]
            );

            if(displace[i / 3] === 0) {
                continue;
            }

            const value = vertexData.value;

            vertex.add(normal.clone().multiplyScalar(value * depth));

            vertices[i + 0] = vertex.x;
            vertices[i + 1] = vertex.y;
            vertices[i + 2] = vertex.z;
        }

        geometry.computeVertexNormals();
        geometry.attributes.position.needsUpdate = true;
    }

    getImageCanvas(image) {
        const canvas = document.createElement('canvas');
        canvas.width = image.width;
        canvas.height = image.height;

        const context = canvas.getContext('2d');
        context.drawImage(image, 0, 0);

        try {
            return context;
        } catch (error) {
            return console.error('Failed to get image canvas:', error);
        }
    }

    getDisplacementValue(imageData, u, v, width, height) {
        const index = (Math.floor(u * (width - 1)) + (Math.floor(v * (height - 1)) * width)) * 4;
        const pixel = (imageData.data[index] + imageData.data[index + 1]) / 3; // Assuming it's grayscale

        return (pixel / 255) - 0.5; // Normalize to [-0.5, 0.5]
    }

    materialInit() {
        const image = this.texture.image;
        const context = this.getImageCanvas(image);

        if (this.greyscale) {
            const imgData = context.getImageData(0, 0, image.width, image.height);
            const pixels = imgData.data;

            for (var i = 0; i < pixels.length; i += 4) {
                let lightness = Math.floor((pixels[i] + pixels[i + 1] + pixels[i + 2]) / 3);

                pixels[i + 0] = lightness;
                pixels[i + 1] = lightness;
                pixels[i + 2] = lightness;
            }

            context.putImageData(imgData, 0, 0);
        }

        this.material = new THREE.MeshPhongMaterial({
            map: new THREE.CanvasTexture(context.canvas),
            // bumpMap: displacementMap,
            // bumpScale: -2,
            // displacementMap: displacementMap,
            // displacementScale: -2,
            // side: THREE.FrontSide
        });
    }

    meshInit(onTextureReady) {
        this.textureInit(this.imageURL, (texture) => {
            this.texture = texture;

            this.sceneInit();
            this.materialInit();
            this.geometryInit(texture.image.width, texture.image.height, function () {
                console.log(this.geometry);
                const mesh = new THREE.Mesh(this.geometry, this.material);
                mesh.scale.x = mesh.scale.y = mesh.scale.z = this.scale;
                this.scene.add(mesh);

                if (onTextureReady) {
                    onTextureReady.call(this);
                }
            });

        });

        //this.download(displacedMesh);
    }

    animate() {
        requestAnimationFrame(this.animate.bind(this));

        this.render();
    }

    render() {
        this.renderer.render(this.scene, this.camera);
    }

    download(mesh) {
        const link = document.createElement('a');
        link.style.display = 'none';
        document.body.appendChild(link);

        function save(blob, filename) {
            link.href = URL.createObjectURL(blob);
            link.download = filename;
            link.click();
        }

        function saveArrayBuffer(buffer, filename) {
            save(new Blob([buffer], {type: 'application/octet-stream'}), filename);
        }

        // Parse the input and generate the STL encoded output
        // const result = exporter.parse( mesh, options );
        //saveArrayBuffer( result, 'box.stl' );
    }

    onResize() {
        this.camera.aspect = this.getAspectRatio();
        this.camera.updateProjectionMatrix();
        this.renderer.setSize(this.container.clientWidth, this.container.clientHeight);
        this.render();
    }
}

let modelViewer = new ModelViewer();

onMounted(() => {

    modelViewer.onMount();

});

</script>

<template>
    <Head title="Create"/>

    <AuthenticatedLayout>
        <div id="wrapper-container">
            <div id="options-container">

            </div>
            <div id="main-container">

            </div>
        </div>
    </AuthenticatedLayout>
</template>

<style scoped>
#wrapper-container {
    position: relative;
    display: block;
    height: calc(100vh - var(--topbar-height));

    --options-container-width: 400px;
    --topbar-height: 4rem;
}

#options-container {
    position: absolute;
    left: 0;
    width: var(--options-container-width);
    top: 0;
    bottom: 0;
    padding: 10px;
}

#main-container {
    position: absolute;
    left: var(--options-container-width);
    right: 0;
    top: 0;
    bottom: 0;

    background: #11e8bb; /* Old browsers */
    background: -moz-linear-gradient(top, #11e8bb 0%, #8200c9 100%); /* FF3.6-15 */
    background: -webkit-linear-gradient(top, #11e8bb 0%, #8200c9 100%); /* Chrome10-25,Safari5.1-6 */
    background: linear-gradient(to bottom, #11e8bb 0%, #8200c9 100%); /* W3C, IE10+, FF16+, Chrome26+, Opera12+, Safari7+ */
    filter: progid:DXImageTransform.Microsoft.gradient(startColorstr='#11e8bb', endColorstr='#8200c9', GradientType=0); /* IE6-9 */
    overflow: hidden;
}
</style>

<style>
.slider {
    -webkit-appearance: none;
    width: 100%;
    height: 15px;
    border-radius: 5px;
    background: #d3d3d3;
    outline: none;
    opacity: 0.7;
    -webkit-transition: .2s;
    transition: opacity .2s;
}

.slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 25px;
    height: 25px;
    border-radius: 50%;
    background: #04AA6D;
    cursor: pointer;
}

.slider::-moz-range-thumb {
    width: 25px;
    height: 25px;
    border-radius: 50%;
    background: #04AA6D;
    cursor: pointer;
}

/* The switch - the box around the slider */
.switch {
    --switch-size: 24px;
    --switch-inset: 4px;

    position: relative;
    display: inline-block;
    width: calc(var(--switch-size) * 1.7);
    height: var(--switch-size);
}

/* Hide default HTML checkbox */
.switch input {
    opacity: 0;
    width: 0;
    height: 0;
}

/* The slider */
.switch-slider {
    position: absolute;
    cursor: pointer;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: #ccc;
    -webkit-transition: .4s;
    transition: .4s;
}

.switch-slider:before {
    position: absolute;
    content: "";
    height: calc(var(--switch-size) - calc(var(--switch-inset) * 2));
    width: calc(var(--switch-size) - calc(var(--switch-inset) * 2));
    left: 4px;
    bottom: 4px;
    background-color: white;
    -webkit-transition: .4s;
    transition: .4s;
}

input:checked + .switch-slider {
    background-color: #2196F3;
}

input:focus + .switch-slider {
    box-shadow: 0 0 1px #2196F3;
}

input:checked + .switch-slider:before {
    -webkit-transform: translateX(calc(var(--switch-size) - calc(var(--switch-inset) * 2)));
    -ms-transform: translateX(calc(var(--switch-size) - calc(var(--switch-inset) * 2)));
    transform: translateX(calc(var(--switch-size) - calc(var(--switch-inset) * 2)));
}

/* Rounded sliders */
.switch-slider.round {
    border-radius: 34px;
}

.switch-slider.round:before {
    border-radius: 50%;
}
</style>
