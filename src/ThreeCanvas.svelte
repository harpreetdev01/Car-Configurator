<script>
    import { onMount, onDestroy } from 'svelte';
    import * as THREE from 'three';
    import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
    import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';
  import { text } from 'svelte/internal';

    let container;
    let scene, camera, renderer, directionalLight, ambientLight, controls, model;

    // Initialize the scene
    onMount(() => {

        // Scene setup
        scene = new THREE.Scene();
        camera = new THREE.PerspectiveCamera(75, container.clientWidth / container.clientHeight, 0.1, 1000);
        camera.position.z = 3.5;

        // Renderer set up
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(container.clientWidth, container.clientHeight);
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 0.4;
        container.appendChild(renderer.domElement);

        // Directional Light
        directionalLight = new THREE.DirectionalLight(0xffffff, 2);
        directionalLight.position.set(0, 5, 5);
        directionalLight.castShadow = true;
        scene.add(directionalLight);

        const pointLight = new THREE.PointLight(0xffffff, 3, 100); // Intensity and distance
        pointLight.position.set(0, 5, 0); // Adjust position based on car location
        // scene.add(pointLight);

        // PMREM Generator for HDR lighting
        const pmremGenerator = new THREE.PMREMGenerator(renderer);
        pmremGenerator.compileEquirectangularShader();

        // Ambient light
        ambientLight = new THREE.AmbientLight(0xfffff, 0.2);
        scene.add(ambientLight);

        // Orbit Controls
        controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.25;
        controls.enableZoom = true;
        controls.minDistance = 3; // Minimum zoom distance
        controls.maxDistance = 3.75; // Maximum zoom distance

        // Env map for reflections
        const rgbeLoader = new RGBELoader();
        rgbeLoader.load('/textures/hdr/rosendal_plains_2_4k.hdr', (texture) => {
            // texture.map = THREE.EquirectangularReflectionMapping;
            // scene.environment = texture;

            const envMap = pmremGenerator.fromEquirectangular(texture).texture;
            scene.background = envMap;
            scene.environment = envMap;
            texture.dispose();
            pmremGenerator.dispose();
            
        // Load GLTF car
        const gltfLoader = new GLTFLoader();
        gltfLoader.load('/models/Car/scene.gltf', (gltf) => {
            model = gltf.scene;

                 // Set up materials to allow for color changes
            model.traverse((node) => {
            if(node.isMesh){
                console.log(node.material); // Log the material for inspection
                if (node.material) {
                    node.material = node.material.clone(); // Clone material for independent changes
                    console.log(node.material.name)
                    if(node.material.name === 'paint'){
                        // node.material.envmap = texture;
                    node.material.envMap = envMap;
                    node.material.envMapIntensity = 3;
                    node.material.roughness = 0.18;
                    node.material.metalness = 1;
                    }
                    
                

                if (node.material.name === 'window') {
                    node.material = new THREE.MeshPhysicalMaterial({
                        color: 0x000000,
                        metalness: 0.1,
                        roughness: 0.05,
                        transmission: 1.0,
                        transparent: true,
                        opacity: 0.87,
                        ior: 1,
                        reflectivity: 0.75,
                        envMapIntensity: 1
                    });
                } 
                // if(node.material.name === ''){
                //     node.material = new THREE.MeshPhysicalMaterial({
                //         color: 0xaaaaaa,
                //         metalness: 0.1,
                //         roughness: 0.05,
                //         transmission: 1.0,
                //         transparent: true,
                //         opacity: 0.7,
                //         ior: 1.5,
                //         reflectivity: 0.8,
                //         envMapIntensity: 1.5
                //     });
                // }
                else {
                    console.warn(`Mesh ${node.name} does not have a material.`);
                }
            }
        }
        });
            

            scene.add(model);
        }, undefined, (error) => {
            console.error('An occured while loading the model:', error);
        });
    });
    
        // Start the animation loop
        animate();
    });

    // Animate the scene
    function animate() {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
    };

    // Clean up when component is destroyed
    onDestroy(() => {
        controls.dispose();
        renderer.dispose();
    });

    // Change color
    function changeModelColor(color, materialName)
    {
        if(model)
        {
            console.log(materialName)
            model.traverse((node) => {
                if(node.isMesh && node.material && node.material.name === materialName)
                {
                    node.material.color.set(color);
                }
            })
        }
    }

</script>

<style>
    /* Basic styling */
    :global(body){
        margin: 0;
        padding: 0;
    }
    .three-container{
        width: 100vw;
        height: 100vh;
        position: fixed;
    }
    .colors
    {
        position: fixed;
        bottom: 20px;
        left: 50%;
        transform: translateX(-50%);

        /* border: 1px solid white; */
    }
    .colors__list
    {
        width: max-content;
        height: max-content;

        display: flex;
        justify-content: center;

        list-style: none;
    }
    .colors__list-item
    {
        cursor: pointer;
    }
    .color
    {
        width: 30px;
        height: 30px;
        
        border-radius: 100%;
        border: 2px solid white;

        margin-left: 15px;
    }
    #red
    {
        background-color: #aa1059;
    }

    #green
    {
        background-color: #093b1e;
    }

    #blue
    {
        background-color: #006ed0;
    }

    #black
    {
        background-color: #000000;
    }
</style>

<div class="three-container" bind:this={container}></div>
<div class="colors">
    <ul class="colors__list">
        <li class="colors__list-item">
            <div class="color" id="red" on:click={() => changeModelColor(0xAA1059, 'paint')}></div>
        </li>
        <li class="colors__list-item">
            <div class="color" id="green" on:click={() =>changeModelColor(0x093B1E, 'paint')}></div>
        </li>
        <li class="colors__list-item">
            <div class="color" id="blue" on:click={() => changeModelColor(0x006ED0, 'paint')}></div>
        </li>
        <li class="colors__list-item">
            <div class="color" id="black" on:click={() => changeModelColor(0x000000, 'paint')}></div>
        </li>
    </ul>
</div>