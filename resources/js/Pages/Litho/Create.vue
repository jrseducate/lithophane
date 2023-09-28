<script setup>
import {ref, onMounted} from 'vue'

import {Head} from '@inertiajs/vue3';
import AuthenticatedLayout from "@/Layouts/AuthenticatedLayout.vue";

import * as THREE from 'three';
import {OrbitControls} from "three/addons/controls/OrbitControls";
import {STLExporter} from 'three/addons/exporters/STLExporter.js';
import {VertexNormalsHelper} from "three/addons/helpers/VertexNormalsHelper";

const exporter = new STLExporter();
const options = {binary: true}

function createElement(options)
{
  const tag = options.tag;
  delete options.tag;

  const element = document.createElement(tag);
  const keys = Object.keys(options);

  for (let i = 0; i < keys.length; i++)
  {
    const key = keys[i];
    const value = options[key];

    switch(key)
    {
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

class ModelViewer
{
  constructor() {
  }

  optionsAddSlider(name, min, max, onUpdate) {
    this.options.appendChild(createElement({
      tag: 'div',
      style: {
        width: '100%',
      },
      className: 'slider-container',
      children: [
        {
          tag: 'label',
          className: 'slider-label',
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
              value: min + Math.floor((max - min) * 0.5),
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

  onMount() {
    this.container = document.getElementById('main-container');
    this.options   = document.getElementById('options-container');
    this.scale     = 0.5;
    this.curvature = 0.5;

    this.init();
  }

  init() {
    this.textureInit((texture) => {
      this.texture = texture;

      this.sceneInit();
      this.rendererInit();

      this.meshInit();
      this.cameraInit(this.geometry.parameters.width);

      this.optionsInit();
      this.bindingsInit();

      this.animate();
    });
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

  geometryInit() {
    this.geometry = new THREE.PlaneGeometry(80, 50, 400, 400);

    this.displaceVertices(this.texture.image); // Adjust the scale as needed
  }

  textureInit(onTextureLoaded) {
    new THREE.TextureLoader().load('http://localhost:8000/images/image.png', onTextureLoaded.bind(this));
  }

  optionsInit() {
    this.optionsAddSlider('Curvature', 0, 100, (e) => this.delay(0, 'set_curvature', () => {
      this.curvature = e.target.value / 100;

      this.textureInit((texture) => {
        this.texture = texture;

        this.sceneInit();
        this.meshInit();
      });
    }));
  }

  bindingsInit() {
    window.addEventListener('resize', this.onResize.bind(this), false);
  }

  calculateUV(x, y, width, height) {
    return {
      u: ((x + (width / 2)) / width),
      v: 1 - ((y + (height / 2)) / height),
    };
  }

  displaceVertices(image) {
    const geometry  = this.geometry;
    const scale     = this.scale;
    const curvature = this.curvature;
    let radius      = this.geometry.parameters.width / 2;

    radius /= curvature * Math.PI;

    geometry.computeVertexNormals();

    const vertices  = geometry.attributes.position.array;
    const normals   = geometry.attributes.normal.array;
    const imageData = this.getImageData(image);

    if (!imageData) {
      return console.error('Failed to access image data.');
    }

    const imageWidth = image.width;
    const imageHeight = image.height;

    const yAxis = new THREE.Vector3(0, 1, 0);

    for (let i = 0; i < vertices.length; i += 3) {
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

      const uv    = this.calculateUV(vertex.x, vertex.y, geometry.parameters.width, geometry.parameters.height);
      const value = this.getDisplacementValue(imageData, uv.u, uv.v, imageWidth, imageHeight);

      if (curvature > 0)
      {
        const center = new THREE.Vector3(
            0,
            vertex.y,
            0
        );
        const direction = (new THREE.Vector3(0, 0, radius)).applyAxisAngle(yAxis, THREE.MathUtils.degToRad((uv.u - 0.5) * 360 * curvature));//.subVectors(vertex, center).normalize().multiplyScalar(radius);
        vertex = (new THREE.Vector3()).addVectors(center, direction).sub(new THREE.Vector3(0, 0, radius));
      }

      vertex.add(normal.clone().multiplyScalar(-value * scale));

      vertices[i + 0] = vertex.x;
      vertices[i + 1] = vertex.y;
      vertices[i + 2] = vertex.z;
    }

    geometry.computeVertexNormals();

    geometry.attributes.position.needsUpdate = true;
  }

  getImageData(image) {
    const canvas  = document.createElement('canvas');
    canvas.width  = image.width;
    canvas.height = image.height;

    const context = canvas.getContext('2d');
    context.drawImage(image, 0, 0);

    try {
      return context.getImageData(0, 0, image.width, image.height);
    }
    catch (error) {
      return console.error('Failed to get displacement image data:', error);
    }
  }

  getDisplacementValue(imageData, u, v, width, height) {
    const index = (Math.floor(u * (width - 1)) + (Math.floor(v * (height - 1)) * width)) * 4;
    const pixel = (imageData.data[index] + imageData.data[index + 1]) / 3; // Assuming it's grayscale

    return (pixel / 255) - 0.5; // Normalize to [-0.5, 0.5]
  }

  materialInit() {
    this.material = new THREE.MeshPhongMaterial({
      map: this.texture,
      // bumpMap: displacementMap,
      // bumpScale: -2,
      // displacementMap: displacementMap,
      // displacementScale: -2,
      // side: THREE.FrontSide
    });
  }

  meshInit() {
    this.materialInit();
    this.geometryInit();

    const mesh = new THREE.Mesh(this.geometry, this.material);

    this.scene.add(mesh);

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
</style>
