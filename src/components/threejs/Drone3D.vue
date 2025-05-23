<template>
    <div ref="threeContainer" class="three-container">

        <!-- 3D 视角控制 UI -->
        <div class="camera-controls">
            <el-radio-group v-model="cameraMode" @change="updateCameraMode">
                <el-radio-button label="free">自由模式</el-radio-button>
                <el-radio-button label="follow">尾随模式</el-radio-button>
                <el-radio-button label="broadcast">自动导播</el-radio-button>
                <el-radio-button label="track">轨迹观察</el-radio-button>
            </el-radio-group>
        </div>
    </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref, defineExpose } from 'vue';
import * as THREE from 'three';
import { FBXLoader } from 'three/examples/jsm/loaders/FBXLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import * as TWEEN from '@tweenjs/tween.js';
const threeContainer = ref(null);
let scene, camera, renderer, controls, airplane;
const cameraMode = ref("free"); // 默认自由模式
let autoBroadcastInterval = null; //自动
let normalTrailGeometry, abnormalTrailGeometry, normalTrailLine, abnormalTrailLine;
let normalTrailVertices = [];
let abnormalTrailVertices = [];
let smoothFactor = 0.2;  // 平滑移动比例


// **初始化 Three.js 场景**
const initScene = () => {
    // 创建场景并设置天空蓝背景
    scene = new THREE.Scene()
    scene.background = new THREE.Color(0xbfdfff)

    // 相机设置
    const aspectRatio = window.innerWidth / window.innerHeight
    camera = new THREE.PerspectiveCamera(90, aspectRatio, 0.1, 1000)
    camera.position.set(0, 10, 20)

    // 渲染器设置 + 阴影开启
    renderer = new THREE.WebGLRenderer({ antialias: true })
    renderer.setSize(window.innerWidth - 480, window.innerHeight - 120)
    renderer.shadowMap.enabled = true
    renderer.shadowMap.type = THREE.PCFSoftShadowMap
    threeContainer.value.appendChild(renderer.domElement)

    // ===== 🌤️ 光照系统 =====
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6)
    scene.add(ambientLight)

    const directionalLight = new THREE.DirectionalLight(0xfff4e6, 0.8)
    directionalLight.position.set(30, 50, 30)
    directionalLight.castShadow = true
    directionalLight.shadow.mapSize.width = 1024
    directionalLight.shadow.mapSize.height = 1024
    scene.add(directionalLight)

    // ===== 🌍 地面 + 阴影 =====
    const ground = new THREE.Mesh(
        new THREE.PlaneGeometry(1000, 1000),
        new THREE.MeshStandardMaterial({ color: 0xeFeFeF })
    )
    ground.rotation.x = -Math.PI / 2
    ground.receiveShadow = true
    scene.add(ground)

    // ===== 🧮 网格参考 =====
    const gridHelper = new THREE.GridHelper(100, 100)
    scene.add(gridHelper)

    // 坐标轴
    const axesHelper = new THREE.AxesHelper(15)
    scene.add(axesHelper)

    // ====== 重点：初始化两条轨迹（正常&异常） ======
    // --- 1) 正常轨迹 (蓝色) ---
    // ===== 1) 正常轨迹：蓝色 =====
    normalTrailGeometry = new THREE.BufferGeometry();
    const normalTrailMaterial = new THREE.LineBasicMaterial({
        color: 0x0000ff,   // 蓝色
        linewidth: 2,
        depthWrite: false,
        polygonOffset: true,
        polygonOffsetFactor: -1,
        polygonOffsetUnits: -1
    });
    normalTrailLine = new THREE.Line(normalTrailGeometry, normalTrailMaterial);
    scene.add(normalTrailLine);

    // ===== 2) 异常轨迹：红色 =====
    abnormalTrailGeometry = new THREE.BufferGeometry();
    const abnormalTrailMaterial = new THREE.LineBasicMaterial({
        color: 0xff0000,   // 红色
        linewidth: 2,
        depthWrite: false,
        polygonOffset: true,
        polygonOffsetFactor: -1,
        polygonOffsetUnits: -1
    });
    abnormalTrailLine = new THREE.Line(abnormalTrailGeometry, abnormalTrailMaterial);
    scene.add(abnormalTrailLine);

    // ===== 🎮 Orbit 控制器 =====
    //添加相机控制参数
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true; // 启用阻尼效果（惯性）
    controls.dampingFactor = 0.05; // 阻尼系数
    controls.screenSpacePanning = true; // 禁用屏幕平移
    controls.minDistance = 4; // 限制最近距离
    controls.maxDistance = 100; // 限制最远距离
    controls.maxPolarAngle = Math.PI / 2; // 限制俯视角度

    // 默认自由模式
    updateCameraMode("free")
}


// **动画循环**
const animate = () => {
    requestAnimationFrame(animate);
    controls.update(); // **更新 OrbitControls**
    renderer.render(scene, camera);
};

const jetPath = new URL('@/assets/models/jet.fbx', import.meta.url).href;

// **加载 FBX 无人机模型**
const loadModel = () => {
    const loader = new FBXLoader();
    loader.load(jetPath, (fbx) => {
        fbx.scale.set(0.5, 0.5, 0.5);
        fbx.position.set(0, 1, 0); // 将无人机模型放置在原点
        //fbx.rotation.set(0, Math.PI / 2, 0);
        //fbx.rotation.set(0, 0, Math.PI / 2);

        // **增强飞机材质**
        fbx.traverse((child) => {
            if (child.isMesh) {
                child.material = new THREE.MeshStandardMaterial({
                    color: 0xff4500, // **橙色，使其更显眼**
                    metalness: 0.7,  // **增加金属反光**
                    roughness: 0.3,  // **适度增加高光**
                    emissive: 0x111111, // **微弱的自发光，使其更明显**
                });
                child.castShadow = true;
                child.receiveShadow = true;
            }
        });

        scene.add(fbx);
        airplane = fbx;
    }, undefined, (error) => {
        console.error('Error loading FBX:', error);
    });
};

// **更新摄像机模式**
const updateCameraMode = (mode) => {
    if (mode === "follow") {
        console.log("📸 切换到尾随模式");
        //followCamera = true; // 启用摄像机跟随
    } else if (mode === "free") {
        console.log("🎥 切换到自由模式");
        //camera.position.set(0, 15, 30);
        //followCamera = false; // 禁用摄像机跟随
    } else if (mode === "broadcast") {
        console.log("📡 启动自动导播模式");
        //followCamera = false;
        startAutoBroadcast();
    } else if (mode === "track") {
        console.log("🔍 轨迹观察模式");
        //followCamera = false;
        adjustCameraForTrackView();
    }
};

// **自动导播模式：根据无人机状态切换视角**
const startAutoBroadcast = () => {
    if (autoBroadcastInterval) clearInterval(autoBroadcastInterval);

    autoBroadcastInterval = setInterval(() => {
        if (!airplane || normalTrailVertices.length < 20) return;

        if (cameraMode.value !== "broadcast") {
            clearInterval(autoBroadcastInterval);
            return;
        }
        // 计算最近 N 个轨迹点的变化趋势
        //const historySize = 20;
        const first = new THREE.Vector3(normalTrailVertices[0], normalTrailVertices[1], normalTrailVertices[2]);
        const last = new THREE.Vector3(
            normalTrailVertices[normalTrailVertices.length - 3],
            normalTrailVertices[normalTrailVertices.length - 2],
            normalTrailVertices[normalTrailVertices.length - 1]
        );

        const dx = Math.abs(last.x - first.x);
        const dz = Math.abs(last.z - first.z);

        let targetPosition;
        if (dx > dz) {
            targetPosition = new THREE.Vector3(40, 20, 0); // **X 方向观察**
        } else if (dz > dx) {
            targetPosition = new THREE.Vector3(0, 20, 40); // **Z 方向观察**
        } else {
            targetPosition = new THREE.Vector3(0, 20, 0); // **俯瞰视角**
        }

        // **平滑移动摄像机**
        camera.position.lerp(targetPosition, 0.8);
        camera.lookAt(airplane.position);
    }, 5000); // 每 5 秒调整一次视角
};


// **更新飞机状态**
const updateAirplaneState = ({ position, rotation, operation_class }) => {
    if (!airplane) return;

    // 初始化未缩放的飞机位置
    //const rawPosition = new THREE.Vector3(position.x * 0.1, position.y * 0.1, position.z * 0.1);

    // 更新飞机位置和旋转（位置乘以0.1）
    airplane.position.set(position.x * 0.1, position.y * 0.1, position.z * 0.1);
    airplane.rotation.set(rotation.x, rotation.y + Math.PI / 2, rotation.z);

    //轨迹点】分两种情况：
    //    如果是 normal，则推入 normalTrailVertices，否则推入 abnormalTrailVertices
    const px = position.x * 0.1;
    const py = position.y * 0.1;
    const pz = position.z * 0.1;

    if (operation_class === "正常") {
        // ------ 正常 => 加到蓝色那条线 ------
        normalTrailVertices.push(px, py, pz);

        normalTrailGeometry.setAttribute(
            'position',
            new THREE.Float32BufferAttribute(normalTrailVertices, 3)
        );
        normalTrailGeometry.needsUpdate = true;

    } else {
        // ------ 异常 => 加到红色那条线 ------
        abnormalTrailVertices.push(px, py, pz);

        abnormalTrailGeometry.setAttribute(
            'position',
            new THREE.Float32BufferAttribute(abnormalTrailVertices, 3)
        );
        abnormalTrailGeometry.needsUpdate = true;
    }

    // 3) 输出 operation_class
    console.log(operation_class);



    //处理视角
    if (cameraMode.value === "follow") {
        // 计算目标位置（飞机后方）
        const offset = new THREE.Vector3(0, 3, -8); // 摄像机位于飞机后上方
        offset.applyQuaternion(airplane.quaternion); // 让偏移方向跟随飞机旋转

        // 计算目标位置
        const targetPosition = airplane.position.clone().add(offset);

        // 使用 Lerp 平滑移动摄像机
        camera.position.lerp(targetPosition, smoothFactor);

        // 更新 OrbitControls 的目标为飞机的位置
        controls.target.copy(airplane.position);
        controls.update();
    }
};

// **优化后的轨迹观察模式：科学计算俯瞰视角**
const adjustCameraForTrackView = () => {
    if (!airplane || normalTrailVertices.length === 0) return;

    // 计算轨迹点的包围盒
    const positions = new Float32Array(normalTrailVertices);
    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.computeBoundingBox();

    const boundingBox = geometry.boundingBox;
    const center = new THREE.Vector3();
    boundingBox.getCenter(center);

    const size = new THREE.Vector3();
    boundingBox.getSize(size);

    // 计算最佳摄像机距离
    const maxDim = Math.max(size.x, size.y, size.z);
    const fov = camera.fov * (Math.PI / 180);
    const distance = maxDim / (2 * Math.tan(fov / 2));

    // **设置 30° 或 45° 斜视角**
    const tiltAngle = Math.PI / 6; // 30° 斜视角（Math.PI / 4 则为 45°）
    const altitude = distance * 1.5;

    // 计算摄像机目标位置（保持视角稳定）
    camera.position.set(
        center.x + Math.sin(tiltAngle) * altitude,
        center.y + altitude,
        center.z + Math.cos(tiltAngle) * altitude
    );
    camera.lookAt(center);

    // **平滑过渡到目标位置**
    new TWEEN.Tween(camera.position)
        .to({ x: camera.position.x, y: camera.position.y, z: camera.position.z }, 1000)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start();

    new TWEEN.Tween(controls.target)
        .to({ x: center.x, y: center.y, z: center.z }, 1000)
        .easing(TWEEN.Easing.Quadratic.Out)
        .start();

    // **轨迹观察模式启用后，不再自动更新**
    cameraMode.value = "track";
};


// **清除轨迹**
const clearTrail = () => {
    // 1. 清空这两条轨迹的顶点数据
    normalTrailVertices = [];
    abnormalTrailVertices = [];

    // 2. 更新正常轨迹几何（让它知道数据长度变为0）
    normalTrailGeometry.setAttribute(
        'position',
        new THREE.Float32BufferAttribute(normalTrailVertices, 3)
    );
    normalTrailGeometry.needsUpdate = true;

    // 3. 更新异常轨迹几何
    abnormalTrailGeometry.setAttribute(
        'position',
        new THREE.Float32BufferAttribute(abnormalTrailVertices, 3)
    );
    abnormalTrailGeometry.needsUpdate = true;
};



// **窗口大小变化**
const onWindowResize = () => {
    if (camera && renderer) {
        const width = window.innerWidth - 480;
        const height = window.innerHeight - 120;
        camera.aspect = width / height;
        camera.updateProjectionMatrix();
        renderer.setSize(width, height);
    }
};

// **Vue 生命周期**
onMounted(() => {
    initScene();
    loadModel();
    animate();
    window.addEventListener('resize', onWindowResize);
});

onUnmounted(() => {
    window.removeEventListener('resize', onWindowResize);
});

defineExpose({ updateAirplaneState, clearTrail });
</script>

<style scoped>
.three-container {
    width: 100%;
    height: 100%;
    position: relative;
    /* 新增定位上下文 */
}

.three-container {
    width: 100%;
    height: 100%;
}

.camera-controls {
    position: absolute;
    bottom: 60px;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(0, 0, 0, 0.5);
    padding: 10px;
    border-radius: 10px;
}
</style>
