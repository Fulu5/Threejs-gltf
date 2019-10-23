<template>
  <div class="hello">
    <!-- loading -->
    <div class="load-bg" id="loadingDiv">
      <div class="load-tex">
        <div class="load-logo">
          <div class="loading-percent">0%</div>
          <h2>
            <span class="loading-bar"></span>
          </h2>
          <p class="Load-Sketch">精雕科技</p>
        </div>
      </div>
    </div>
    <!-- right-buttons -->
    <div class="right-buttons">
      <ul style="list-style: none;">
        <li>
          <img src="../assets/img/btn-cf.png" alt="explode.icon">
          <div class="line-content" style="display: block;">
            <input type="range" min="0" max="100" value="0" class="slider" id="explodeRange">
          </div>
        </li>
        <li>
          <img src="../assets/img/btn-light.png" alt="light.icon">
          <div class="line-content" style="display: block;">
            <input type="range" min="0" max="100" value="0" class="slider" id="lightIndensity">
          </div>
        </li>
      </ul>
    </div>
    <!-- down buttons -->
    <div class="down-buttons">
      <ul style="list-style: none;">
        <li class="clarity" @click="enableClarity" v-show="operating">
          <img src="../assets/img/btn-clarity.png" alt="clarity.icon">
        </li>
        <li class="notClarity" @click="enableNotClarity" v-show="operating">
          <img src="../assets/img/btn-notClarity.png" alt="notClarity.icon">
        </li>
        <li class="show" @click="showAllMesh" v-show="operating">
          <img src="../assets/img/btn-show.png" alt="show.icon">
        </li>
        <li class="showHide" @click="switchShowHide" v-show="operating">
          <img src="../assets/img/btn-showHide.png" alt="showHide.icon">
        </li>
        <li class="hide" @click="hideSelectObject" v-show="operating">
          <img src="../assets/img/btn-hide.png" alt="hide.icon">
        </li>
        <li class="move" @click="enableDrag" v-show="operating">
          <img src="../assets/img/btn-move.png" alt="move.icon">
        </li>
        <li class="select" @click="enableOperating">
          <img src="../assets/img/btn-handle.png" alt="select.icon">
        </li>
      </ul>
    </div>

    <!-- content -->
    <div id="WebGL-output" style="touch-action: none;"></div>
  </div>
</template>

<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { DragControls } from "three/examples/jsm/controls/DragControls";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
// import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";
// import { DDSLoader } from "three/examples/jsm/loaders/DDSLoader";

export default {
  name: "HelloWorld",
  props: {
    msg: String
  },
  data() {
    return {
      scene: "",
      camera: "",
      renderer: "",
      controls: "",
      dragControls: "",
      clickObjects: [],
      _curObj: "",
      operating: false
    };
  },
  created() {},
  mounted() {
    this.initTHREE();
    this.animate();
    this.enableLightInput();
    this.enableExplode();
  },
  methods: {
    initTHREE() {
      this.scene = new THREE.Scene();
      // this.scene.position.set(0, -2, 0);
      this.camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 1, 10000);
      this.camera.position.y = 200;
      this.camera.position.z = 1000;
      this.camera.lookAt(this.scene.position);
      this.renderer = new THREE.WebGLRenderer({ antialias: true });
      // 背景色
      this.renderer.setClearColor(0xeeeeee);
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      var renderCanvas = this.renderer.domElement;
      document.getElementById("WebGL-output").append(renderCanvas);

      // controls
      this.controls = new OrbitControls(this.camera, renderCanvas);
      this.controls.rotateSpeed = 0.5;
      this.controls.enableDamping = true;
      // controls.maxPolarAngle = Math.PI / 2;

      this.dragControls = new DragControls(this.clickObjects, this.camera, renderCanvas);
      this.dragControls.enabled = false;
      this.dragControls.deactivate();

      window.addEventListener("resize", this.onWindowResize, false);

      // lights
      this.scene.add(new THREE.AmbientLight(0x0c0c0c));
      var pointLight = new THREE.PointLight(0xffffff, 1);
      this.camera.add(pointLight);
      this.scene.add(this.camera);

      // axes
      // var axesHelper = new THREE.AxesHelper(1000);
      // this.scene.add(axesHelper);

      this.raycaster = new THREE.Raycaster();
      this.mouse = new THREE.Vector2();
      window.addEventListener("mousemove", this.onMouseMove, false);

      // 加载 gltf
      var that = this;
      var loader = new GLTFLoader();
      console.time("gltf load");
      // gltf
      loader.load("../../static/example/gr200.gltf", that.onGLTF, that.onProgress, that.onError);
      // loader.load("../../static/obj/01.gltf", that.onGLTF, that.onProgress, that.onError);
      // draco gltf
      // loader.load("../../public/static/gltf/CesiumMan.gltf", that.onGLTF, that.onProgress, that.onError);
    },
    // 加载进度
    onProgress(xhr) {
      if (xhr.lengthComputable) {
        var count = (xhr.loaded / xhr.total) * 100;
        var percentText = document.getElementsByClassName("loading-percent")[0];
        var percentBar = document.getElementsByClassName("loading-bar")[0];
        percentText.innerHTML = Math.round(count, 2) + "%";
        percentBar.style.width = count + "%";
      }
    },
    // 加载错误
    onError(xhr) {
      console.log("GLTFLoader().load", xhr);
    },
    // 加载成功回调
    onGLTF(gltf) {
      this.scene.add(gltf.scene);
      document.getElementById("loadingDiv").style.display = "none";
      console.timeEnd("gltf load");

      var that = this;
      // 模型的中心点坐标，爆炸中心
      var modelWorldPs = this.getModelCenter(gltf.scene);

      gltf.scene.traverse(function(value) {
        if (value.isMesh) {
          // 每个mesh的中心点
          var worldPs = that.getMeshCenter(value);
          if (isNaN(worldPs.x)) return;
          // 爆炸方向
          value.worldDir = new THREE.Vector3().subVectors(worldPs, modelWorldPs).normalize();
          // 保存初始坐标
          value.userData.oldPs = value.getWorldPosition(new THREE.Vector3());
          // var box = new THREE.Box3().setFromObject(value);
          // value.direct = box
          //   .getCenter(new THREE.Vector3())
          //   .sub(center)
          //   .clone()
          //   .multiplyScalar(0.01);

          that.clickObjects.push(value);
        }
      });
      // document.querySelector("#explodeRange").addEventListener("input", function(evt) {
      //   var scalar = this.value * 10;
      //   gltf.scene.traverse(function(value) {
      //     if (!value.isMesh || !value.worldDir) return;
      //     value.position.copy(new THREE.Vector3().copy(value.userData.oldPs).add(new THREE.Vector3().copy(value.worldDir).multiplyScalar(scalar)));
      //   });
      // });
    },
    getModelCenter(value) {
      var modelBox3 = new THREE.Box3();
      modelBox3.expandByObject(value);
      return new THREE.Vector3().addVectors(modelBox3.max, modelBox3.min).multiplyScalar(0.5);
    },
    getMeshCenter(value) {
      var meshBox3 = new THREE.Box3();
      meshBox3.setFromObject(value);
      return new THREE.Vector3().addVectors(meshBox3.max, meshBox3.min).multiplyScalar(0.5);
    },
    animate() {
      var that = this;
      requestAnimationFrame(function() {
        that.animate();
      });
      this.controls.update(); // only required if controls.enableDamping = true, or if controls.autoRotate = true
      this.render();
    },
    render() {
      this.renderer.render(this.scene, this.camera);
    },
    onWindowResize() {
      this.camera.aspect = window.innerWidth / window.innerHeight;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
    },
    // 亮度
    enableLightInput() {
      var that = this;
      document.querySelector("#lightIndensity").addEventListener("input", function(evt) {
        that.camera.children[0].power = (1 + this.value * 0.01) * 4 * Math.PI;
      });
    },
    // 拆分
    enableExplode() {
      var that = this;
      document.querySelector("#explodeRange").addEventListener("input", function(evt) {
        var scalar = this.value * 10;
        console.log(scalar);
        that.clickObjects.forEach(function(value) {
          if (!value.isMesh || !value.worldDir) return;
          // var worldPos = value.getWorldPosition(new THREE.Vector3());
          // worldPos.add(value.direct.clone().multiplyScalar(scalar));
          // var localPos = value.parent.worldToLocal(worldPos);
          // value.position.copy(localPos);
          value.position.copy(new THREE.Vector3().copy(value.userData.oldPs).add(new THREE.Vector3().copy(value.worldDir).multiplyScalar(scalar)));
        });
      });
    },
    // 操作
    enableOperating() {
      if (this.operating) {
        document.getElementById("WebGL-output").removeEventListener("mousedown", this.onDocumentClick, false);
        this.showAllMesh();
        this.cancelSelectObject();
        this.operating = false;
      } else {
        this.operating = true;
        document.getElementById("WebGL-output").addEventListener("mousedown", this.onDocumentClick, false);
      }
    },
    onDocumentClick(event) {
      event.preventDefault();
      var obj = this.clickSingleObject(event);
      if (obj) {
        this.setSelectObject(obj);
      } else {
        this.cancelSelectObject();
      }
    },
    clickSingleObject(event) {
      if (event.touches) {
        //兼容移动端
        event = event.touches[0];
      }
      var raycaster = new THREE.Raycaster();
      var mouse = new THREE.Vector2();
      mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
      raycaster.setFromCamera(mouse, this.camera);
      var intersects = raycaster.intersectObjects(this.clickObjects, true);
      if (intersects.length == 0) return;
      var objectTemp = intersects[0].object;
      objectTemp.faceIndex = intersects[0].faceIndex;
      return objectTemp;
    },
    // 设置选中状态
    setSelectObject(object) {
      if (object.isMesh) {
        if (this._curObj && this._curObj !== object) {
          this.cancelSelectObject();
        }

        this._curObj = object;
        if (!this._curObj.originalMaterial) {
          this._curObj.originalMaterial = this._curObj.material.clone(true);
        }

        this._curObj.material.color.set(0xff0000);
      }
    },
    //取消选中的对象
    cancelSelectObject() {
      if (this._curObj) {
        if (this._curObj.transparent) {
          this.transparentTransparency();
        } else {
          this._curObj.material = this._curObj.originalMaterial.clone(true);
          this._curObj = null;
        }
      }
    },
    // 透明
    transparentTransparency() {
      if (this._curObj.originalMaterial instanceof Array) {
        var arrayTemp = [];
        var array = this._curObj.originalMaterial.clone(true);
        var clMat;
        for (var i = 0; i < array.length; i++) {
          clMat = array[i].clone(true);
          clMat.transparent = true;
          clMat.opacity = 0.3;
          clMat.uOpacityFactor = 0.3;
          clMat.needsUpdate = true;
          arrayTemp.push(clMat);
        }
        this._curObj.material = arrayTemp;
      } else {
        this._curObj.material = this._curObj.originalMaterial.clone(true);
        this._curObj.material.transparent = true;
        this._curObj.material.opacity = 0.3;
        this._curObj.material.uOpacityFactor = 0.3;
        this._curObj.material.needsUpdate = true;
      }
      this._curObj.transparent = true;
      this._curObj = null;
    },
    // 取消透明
    cancelTransparentTransparency() {
      if (this._curObj && this._curObj.transparent) {
        this._curObj.transparent = false;
        this._curObj.material = this._curObj.originalMaterial.clone(true);
        this._curObj = null;
      }
    },
    // 移动
    enableDrag() {
      if (this.operating) {
        this.controls.enabled = false;
        this.dragControls.enabled = true;
        this.dragControls.activate();
      } else {
        this.dragControls.enabled = false;
        this.dragControls.deactivate();
        this.controls.enabled = true;
      }
    },
    setDragControlsEventListener() {
      var that = this;
      this.dragControls.addEventListener("dragstart", this.disableOrbitControls());
      this.dragControls.addEventListener("dragend", this.enabledOrbitControls());
    },
    disableOrbitControls() {
      this.controls.enabled = false;
      this.setSelectObject
    },
    enabledOrbitControls() {
      this.controls.enabled = true;
    },
    // 隐藏
    hideSelectObject() {
      if (this._curObj) {
        this._curObj.visible = false;
        this.cancelSelectObject();
      } else {
        alert("请选择模型");
      }
    },
    // 显隐互换
    switchShowHide() {
      this.clickObjects.forEach(function(mesh) {
        mesh.visible = !mesh.visible;
      });
    },
    // 全显
    showAllMesh() {
      this.clickObjects.forEach(function(mesh) {
        mesh.visible = true;
        if (mesh.originalMaterial) {
          mesh.material = mesh.originalMaterial;
        }
      });
    },
    // 不透明
    enableNotClarity() {
      if (this._curObj && this._curObj.material.transparent) {
        this._curObj.material = this._curObj.originalMaterial.clone(true);
        this._curObj.material.transparent = false;
        this._curObj.material.opacity = 1;
      }
    },
    // 透明
    enableClarity() {
      if (this._curObj) {
        if (this._curObj.transparent) {
          this.cancelTransparentTransparency();
        } else {
          this.transparentTransparency();
        }
      } else {
        alert("请选择模型");
      }
    }
  }
};
</script>

<style scoped>
@import "../assets/css/index.css";
</style>
