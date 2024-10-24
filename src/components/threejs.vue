<template>
  <div>
    <div style="display: flex; gap: 50px">
      <input type="file" @change="onFileChange" accept=".gltf,.glb" multiple />
      <div>名称：{{ nodeName }}</div>
      <div>坐标：{{ coordinateDisplay }}</div>
      <div>
        <button @click="disappearWallfun(2217802)">隐藏墙壁</button>
        <button @click="disappearWallfun(1701952)">隐藏柱子</button>
        <button @click="disappearWallfun(2239507)">隐藏窗户</button>
      </div>
      <div>
        <button @click="moveElement('R')">右</button>
        <button @click="moveElement('L')">左</button>
        <button @click="moveElement('U')">⬆️</button>
        <button @click="moveElement('D')">⬇️</button>
      </div>
    </div>
    <div ref="canvasContainer" class="canvas-container"></div>
    <div id="info">
      <li v-for="(it, index) in elementInfo" :key="index">{{ it.propertyname }}: {{ it.value }}</li>
    </div>
  </div>
</template>
<script src="https://cdnjs.cloudflare.com/ajax/libs/tween.js/18.6.4/tween.umd.js"></script>
<script>
// 引入three.js
import * as THREE from "three";
// 导入轨道控制器
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { Color } from "three";
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass';
import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import axios from 'axios';

export default {
  name: "ThreeJs",
  data() {
    return {
      scene: new THREE.Scene(),
      camera: new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000),
      renderer: new THREE.WebGLRenderer({ antialias: true }),
      controls: null,
      mouse: new THREE.Vector2(),  // 初始化鼠标向量
      raycaster: new THREE.Raycaster(), // 初始化射线检测器
      nodeName: "",
      coordinateDisplay: "",
      selectedObject: null,
      outlinePass: null,
      composer: null,
      renderPass: null,
      axesHelper: new THREE.AxesHelper(5), //增加坐标轴
      object: null,  // 加载模型
      gltf: null,
      isDisappearWall: false,
      elementDetail: null,  // 点击事件返回元素详情
      highLightPoint: null,   // 点击位置高亮点
      elementInfo:[],         // 点击元素详情信息
      clickedObject:null,     // 点击后选中物体
      sphere:null             // 球体
    };
  },
  methods: {
    initScene(object) {
      //添加辅助线
      // this.scene.add(new THREE.AxesHelper(20));
      this.scene.background = new Color('#191919');

      // 计算包围盒
      const box = new THREE.Box3().setFromObject(object);
      const size = box.getSize(new THREE.Vector3()).length();
      let centerOringinal = box.getCenter(new THREE.Vector3());

      console.log(78,box);

      // 获取中心点坐标

      let centerNow = {x:0,y:0,z:0};



      centerNow.x = this.calMid(box.min.x,box.max.x);
      centerNow.y = this.calMid(box.min.y,box.max.y);
      centerNow.z= this.calMid(box.min.z,box.max.z);

      console.log(87,centerNow);


      object.position.sub(centerOringinal);      // 调整模型的位置使其中心对齐到 (0, 0, 0)
      object.rotation.set(270, 0, 0); // 初始化模型旋转

      this.camera.near = size / 100;
      this.camera.far = size * 100;
      this.camera.fov = 60; // 调整视野


      this.camera.position.copy(centerOringinal);

      // 先固定初始化相机位置
      this.camera.position.x = 8
      this.camera.position.y = 115;
      this.camera.position.z = 195;

      this.camera.lookAt(centerOringinal);
      this.camera.updateProjectionMatrix();
      console.log(72, this.camera);
      console.log(73, size);
      this.scene.add(this.camera);

      this.renderer.setClearColor(0xcccccc);
      this.renderer.setPixelRatio(window.devicePixelRatio);

      let width = window.innerWidth;
      let height = window.innerHeight;
      this.renderer.setSize(width, height);

      this.$refs.canvasContainer.appendChild(this.renderer.domElement);

      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.screenSpacePanning = true; // 不支持平移
      // 设置自动旋转
      // this.controls.autoRotate = true;

      const ambientLight = new THREE.AmbientLight('#fff', 2); // 环境光
      this.scene.add(ambientLight);

      const directionalLight = new THREE.DirectionalLight('#fff', 1);
      directionalLight.position.set(1, 1, 1).normalize();
      this.scene.add(directionalLight);

      this.composer = new EffectComposer(this.renderer);
      this.renderPass = new RenderPass(this.scene, this.camera);
      this.composer.addPass(this.renderPass);

      this.outlinePass = new OutlinePass(new THREE.Vector2(window.innerWidth, window.innerHeight), this.scene, this.camera);
      this.outlinePass.visibleEdgeColor.set(0xffff00); // 高亮颜色
      this.composer.addPass(this.outlinePass);

      this.animate();

      this.scene.add(object);
    },
    animate() {
      this.controls.update(); // 更新控制器
      requestAnimationFrame(this.animate);

      // 添加射线检测
      this.raycaster.setFromCamera(this.mouse, this.camera);
      // 设置当前选中的物体（如果有）
      this.outlinePass.selectedObjects = this.selectedObject ? [this.selectedObject] : [];
      this.composer.render();
    },
    onFileChange(event) {
      const file = event.target.files[0];
      console.log(115, event.target.files);
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const contents = e.target.result;
          this.loadGLTFModel(contents);
        };
        reader.readAsArrayBuffer(file);
      }
    },

    loadGLTFModel(buffer) {
      const loader = new GLTFLoader();
      loader.parse(buffer, '', (gltf) => {
        this.gltf = gltf;

        // gltf.scene.position={x:0,y:0,z:0}
        console.log(132, '当前位置：', gltf.scene.position);
        this.initScene(gltf.scene);
        this.object = gltf.scene
        console.log(136, 'camera:', gltf.camera);
        // 设置鼠标事件
        window.addEventListener('click', this.handleClick);

        // 监听窗口变化
        window.addEventListener("resize", this.handleResize);

        // 监听鼠标滑动事件

      });
    },

    handleResize() {
      let width = window.innerWidth;
      let height = window.innerHeight;

      // 重置渲染器宽高比
      this.renderer.setSize(width, height);
      // 重置相机宽高比
      this.camera.aspect = width / height;
      // 更新相机的投影矩阵
      this.camera.updateProjectionMatrix();

    },
    handleClick(event) {

      // console.log(171, 'camera.position: ', this.camera.position.x, this.camera.position.y, this.camera.position.z);
      // 获取鼠标位置并转换为标准化设备坐标
      this.mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      this.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
      this.restoreHighlight();

      // 使用射线检测
      this.raycaster.setFromCamera(this.mouse, this.camera);
      const intersects = this.raycaster.intersectObjects(this.scene.children, true);

      
      if (intersects.length > 0) {
        this.clickedObject = intersects[0].object;
        let id = this.clickedObject.name;
        console.log('选中物体name:', id);

        this.nodeName = this.clickedObject.material.name;
        this.selectedObject = this.clickedObject;
        const intersectedPoint = intersects[0].point;
        this.coordinateDisplay = `(x轴：${intersectedPoint.x}, y轴：${intersectedPoint.y}, z轴：${intersectedPoint.z})`;


        // 在点击点加一个球
        // this.addGeometry(intersectedPoint.x,intersectedPoint.y,intersectedPoint.z);


        // 获取透明度
        this.clickedObject.material.transparent = true;
        this.clickedObject.material.opacity == 1 ? this.clickedObject.material.opacity = 0.1 : this.clickedObject.material.opacity = 1
        // 设置透明度为0.5 ,1 是不透明，0 是透明

        

        // 获取元素详情
        // this.getDetail(id);      

      } else {
        this.selectedObject = null;
      }
    },


    //	选中物体高亮
    highlight(object) {
      console.log('选中物体高亮ID：', object.id)
      object.material.emissive = new THREE.Color(0xffff00);
    },
    // 高亮恢复
    restoreHighlight() {
      // 遍历子对象，恢复未高亮
      this.scene.traverse((child) => {
        if (child.isMesh) {


          child.material.emissive = new THREE.Color(0x000000); // 恢复原颜色
        }
      });
    },

    // 移动选中物体
    moveElement(type){
      // this.object.position=new THREE.Vector3(0, 0, 0); // 对齐模型
      console.log('将要移动选中物体：', this.clickedObject);
      console.log('选中物体position：', this.clickedObject.position);
      console.log('选中物体几何信息max：', this.clickedObject.geometry.boundingBox.max);
      console.log('选中物体几何信息min：', this.clickedObject.geometry.boundingBox.min);
      let result = this.calcElement(this.clickedObject.geometry.boundingBox.max,this.clickedObject.geometry.boundingBox.min);
      console.log('选中物体几何信息result：', result);
      if(type == "R"){
        if(result.target =='x'){
          this.clickedObject.position=new THREE.Vector3(
          this.clickedObject.position.x+1, this.clickedObject.position.y, this.clickedObject.position.z); 
        }else{
          this.clickedObject.position=new THREE.Vector3(
          this.clickedObject.position.x, this.clickedObject.position.y, this.clickedObject.position.z+1); 
        }  
      }else if(type == "L"){
        if(result.target =='x'){
          this.clickedObject.position=new THREE.Vector3(
          this.clickedObject.position.x-1, this.clickedObject.position.y, this.clickedObject.position.z); 
        }else{
          this.clickedObject.position=new THREE.Vector3(
          this.clickedObject.position.x, this.clickedObject.position.y, this.clickedObject.position.z-1); 
        }
      }else if(type == "U"){
        this.clickedObject.position=new THREE.Vector3(
          this.clickedObject.position.x, this.clickedObject.position.y+1, this.clickedObject.position.z); 
      }else{
        this.clickedObject.position=new THREE.Vector3(
          this.clickedObject.position.x, this.clickedObject.position.y-1, this.clickedObject.position.z); 
      }
    },
    // 计算墙体走向
    calcElement(maxObj,minObj){
      let result={x:0,y:0,z:0};
      result.x= maxObj.x - minObj.x;
      result.y= maxObj.y - minObj.y;
      result.z= maxObj.z - minObj.z;

      let maxValue = result.x;
      let maxVariable = 'x';

      // if (result.y > maxValue) {
      //   maxValue = result.y;
      //   maxVariable = 'y';
      // }

      if (result.z > maxValue) {
        maxValue = result.z;
        maxVariable = 'z';
      }
      return {
        result:result,
        target: maxVariable,
        maxValue:maxValue
      };
    },

    //  添加一个球体
    // addGeometry(x,y,z){
    //   let geometry = new THREE.SphereGeometry(5,5,5);
    //   let material = new THREE.MeshBasicMaterial({color:'#ff0000'});

    //   this.sphere = new THREE.Mesh(geometry,material);
    //   this.sphere.position = new THREE.Vector3((x,y,z));

    //   console.error(306,this.sphere);
    //   this.scene.add(this.sphere);
    // },


    // 清空模型中同类元素
    disappearWallfun(id) {
      console.log(231, this.isDisappearWall, id, this.object);

      // 使用axios调用通过id查询接口
      let url = '/getSameCategoryElement?id=' + id;
      axios.get(url, {
        baseURL: 'http://localhost:3000',
        headers: { 'Content-Type': 'application/json' }
      }).then(response => {
        console.log('同类物体：', response.data.data);
        let sameCategory = response.data.data;
        if (this.isDisappearWall == false) {
          this.object.children.forEach(it => {
            if (sameCategory.includes(it.name)) {
              it.material.visible = false;
            }
          });
        } else {
          this.object.children.forEach(it => {
            if (sameCategory.includes(it.name)) {
              it.material.visible = true;
            }
          });
        }
      }).catch(error => {
        console.error(error);
      });

      this.isDisappearWall == true ? this.isDisappearWall = false : this.isDisappearWall = true;

    },

    // 设置相机位置
    setCamera(event, intersects) {

      console.log(305, this.camera);
      let oldPosition = this.camera.position; // 当前相机位置

      let clickedObject = intersects[0].object; // 选中物体
      let boundingSphere = clickedObject.geometry.boundingSphere;// 获取模型的boundingSphere对象
      let targetPosition = intersects[0].point; // 目标位置，即点击元素的位置
      console.log(310, '选中元素信息：', clickedObject.point);

      console.log(326, '相机位置：', this.camera.position.x, this.camera.position.y, this.camera.position.z);

    },

    // API-获取元素详情接口
    getDetail(id) {
      this.elementInfo =[];

      // 使用axios调用通过id查询接口
      let url = '/get?id=' + id;
      axios.get(url, {
        baseURL: 'http://localhost:3000',
        headers: { 'Content-Type': 'application/json' }
      }).then(response => {

        this.elementDetail = response.data.data;
        // for(let i=0;i<this.elementDetail.length;i++){
        for (let i = 0; i <this.elementDetail.length ; i++) { // 象征性输出几个即可。
          if (this.elementDetail[i] != undefined || this.elementDetail[i].value != null || this.elementDetail[i].value.length > 0) {
            let it = this.elementDetail[i];
            this.elementInfo.push(it);
            this.$set(this.elementInfo, this.elementInfo.length, it);
            
            console.log(`${it.category},${it.floor},${it.modelName} ---- ${it.propertyname} : ${it.value == null ? '无' : it.value}`);
          }
        }
      }).catch(error => {
        console.error(error);
      });
    },
    
    
    calMid(A,B){
      let a,b = 0 ;
      a= A;
      b= B;
      if(A<=0){a=-A}
      if(B<=0){b=-B}
      return (a+b)/2;
    },

    // 移动相机聚焦点击元素
    forcusObject(clickedObject){
      // 切换相机视角到点击的模型
      const targetPosition =new THREE.Vector3().copy(clickedObject.position).add(new THREE.Vector3(0.5, 0.5, 0.5));
      
      // const targetPosition = new THREE.Vector3(clickedObject.position);
      // // .add(new THREE.Vector3(0, 20, 30));
      // const targetLookAt = clickedObject.position;
      const targetLookAt = clickedObject.position.clone();
      
      // // 保存当前相机的位置和朝向
      const startPosition = this.camera.position.clone();
      // const startLookAt = new THREE.Vector3();
      // this.camera.getWorldDirection(startLookAt);1
      // this.camera.position.copy(targetPosition);
      this.camera.position.copy(targetPosition);
      this.camera.lookAt(targetLookAt);
    
    },

    


  },
};
</script>
<style scoped></style>
