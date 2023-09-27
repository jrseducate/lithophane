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

class ModelViewer
{
  constructor() {
    this.scene = new THREE.Scene();
    this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    this.renderer = new THREE.WebGLRenderer({antialias: true, alpha: true});
  }

  onMount() {
    this.scale = 0.5;
    this.curvature = 1;

    this.sceneInit();
    this.rendererInit();
    this.geometryInit();
    this.displacementMapInit();

    function onWindowResize() {
      this.camera.aspect = window.innerWidth / window.innerHeight;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      this.render();
    }

    window.addEventListener('resize', onWindowResize.bind(this), false);

    this.camera.position.z = this.geometry.parameters.width;

    const controls = new OrbitControls(this.camera, this.renderer.domElement)
    controls.screenSpacePanning = true //so that panning up and down doesn't zoom in/out
    //controls.addEventListener('change', render)
  }

  sceneInit() {
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
    this.renderer.setSize(window.innerWidth, window.innerHeight);
    document.getElementById('main-container').appendChild(this.renderer.domElement);
  }

  geometryInit() {
    this.geometry = new THREE.PlaneGeometry(80, 50, 400, 400);
  }

  displacementMapInit() {
    this.displacementMap = new THREE.TextureLoader().load('http://localhost:8000/images/image.png', this.textureLoad.bind(this));
  }

  calculateUV(x, y, width, height) {
    return {
      u: ((x + (width / 2)) / width),
      v: 1 - ((y + (height / 2)) / height),
    };
  }

  displaceVertices(displacementMap) {
    const geometry  = this.geometry;
    const scale     = this.scale;
    const curvature = this.curvature;
    let radius    = this.geometry.parameters.width / 2;

    radius /= curvature * Math.PI;

    debugger;

    geometry.computeVertexNormals();

    const vertices              = geometry.attributes.position.array;
    const normals               = geometry.attributes.normal.array;
    const displacementImageData = this.getDisplacementImageData(displacementMap);

    if (!displacementImageData) {
      return console.error('Failed to access displacement map image data.');
    }

    const imageWidth = displacementMap.image.width;
    const imageHeight = displacementMap.image.height;

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
      const value = this.getDisplacementValue(displacementImageData, uv.u, uv.v, imageWidth, imageHeight);

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

  getDisplacementImageData(displacementMap) {
    const canvas  = document.createElement('canvas');
    canvas.width  = displacementMap.image.width;
    canvas.height = displacementMap.image.height;

    const context = canvas.getContext('2d');
    context.drawImage(displacementMap.image, 0, 0);

    try {
      return context.getImageData(0, 0, displacementMap.image.width, displacementMap.image.height);
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

  textureLoad(texture) {
    let displacementMap = this.displacementMap;

    displacementMap.image       = texture.image; // Store the loaded image
    displacementMap.needsUpdate = true; // Update the texture

    this.geometry.computeVertexNormals();

    this.displaceVertices(displacementMap); // Adjust the scale as needed

    const displacedMesh = new THREE.Mesh(this.geometry, new THREE.MeshPhongMaterial({
      map: texture,
      // bumpMap: displacementMap,
      // bumpScale: -2,
      // displacementMap: displacementMap,
      // displacementScale: -2,
      // side: THREE.FrontSide
    }));

    this.scene.add(displacedMesh);

    //this.download(displacedMesh);
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

  animate() {
    requestAnimationFrame(this.animate.bind(this));

    this.render();
  }

  render() {
    this.renderer.render(this.scene, this.camera);
  }
}

let modelViewer = new ModelViewer();

onMounted(() => {

  modelViewer.onMount();

});

modelViewer.animate();
</script>

<template>
  <Head title="Create"/>

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
  background: -moz-linear-gradient(top, #11e8bb 0%, #8200c9 100%); /* FF3.6-15 */
  background: -webkit-linear-gradient(top, #11e8bb 0%, #8200c9 100%); /* Chrome10-25,Safari5.1-6 */
  background: linear-gradient(to bottom, #11e8bb 0%, #8200c9 100%); /* W3C, IE10+, FF16+, Chrome26+, Opera12+, Safari7+ */
  filter: progid:DXImageTransform.Microsoft.gradient(startColorstr='#11e8bb', endColorstr='#8200c9', GradientType=0); /* IE6-9 */
  overflow: hidden;
}
</style>
