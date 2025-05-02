<script>

    
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
    import { Lensflare, LensflareElement } from 'three/examples/jsm/objects/Lensflare.js';
    import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
    
    import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
    import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
    
    import * as THREE from 'three';
    import { TextureLoader, MathUtils, Raycaster, Vector2, Vector3, Scene, Fog, PerspectiveCamera, WebGLRenderer, Color, AmbientLight, DirectionalLight, SphereGeometry, MeshBasicMaterial, Mesh, PointLight, MeshStandardMaterial, BoxGeometry, PlaneGeometry, ShaderMaterial } from 'three';

    import { onMount } from 'svelte';

    import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
    import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
    import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass.js';

    import * as SkeletonUtils from 'three/addons/utils/SkeletonUtils.js';

    let controlsEnabled = false;
    let darknessEnabled = true;

    const oceanDepth = 200;
    const sunPhasePercentage = 0.2;
    const oceanPhasePercentage = 0.5;
    const submarinePhasePercentage = 0.3;

    const scrollWindowHeight = 5_000;


    let loadedCount = 0;
    const MAX_LOADED = 6; // 3 solar flare pngs, 1 font, 1 submarine glb, 1 tv text texture png, 

    let submarineScene = null;
    let submarineAnimations = null;
    let lensflare = null;
    let textFont = null;
    let tvTxtTexture = null
    function loadStuff() {
        
        const gltfLoader = new GLTFLoader();
        const fontLoader = new FontLoader();
        const textureLoader = new TextureLoader();

        lensflare = new Lensflare();
        const flareTextures = [
            { texture: "/img/solar/mainFlare.png", size: 450, strength: 0.2, color: new Color(0x856319) },
            { texture: "/img/solar/orangeFlareM.png", size: 128, strength: 0.3, color: new Color(0x856319) },
            { texture: "/img/solar/orangeFlareS.png", size: 64, strength: 0.4, color: new Color(0x856319) }
        ];

   
        flareTextures.forEach(({ texture, size, strength, color }) => {
            lensflare.addElement(new LensflareElement(
                textureLoader.load(texture),
                size,
                strength,
                color
            ));
            loadedCount++;
        });
        
        fontLoader.load('/font.json', function (font) {
            textFont = font;
            loadedCount++;
        });
        
        // gltfLoader.load('/models/toad.glb', (gltf) => {
        gltfLoader.load('/models/Submarine.glb', (gltf) => {
            submarineScene = SkeletonUtils.clone(gltf.scene);
            submarineAnimations = gltf.animations.slice();

            console.log(submarineScene)


            // console.log("submarineAnimations")
            // console.log(submarineAnimations)



            loadedCount++;
        });
        
        textureLoader.load('/img/research.png', (texture) => {
        // textureLoader.load('/img/geo/rockbanner.png', (texture) => {
            tvTxtTexture = texture;
            loadedCount++;
        });





    }




    const title = "Microplastics";
    const textDepths = [0.3, 0.6, 0.9];
    const textLabels = ['30% Below Water', '60% Below Water', 'submarine'];



    function updatePosition(scroll) {
    
        const padding = 0.2;
        const calcScroll = Math.min(1, Math.max(0, (scroll-sunPhasePercentage) / (oceanPhasePercentage)));
        
        if (calcScroll < this.depth - padding || calcScroll > this.depth + padding)
            return;
        
        const movePercent = (calcScroll - (this.depth - padding)) / ((this.depth + padding) - (this.depth - padding));
        
        if (this.isSubmarine) {
            this.mesh.position.x = this.initialX - (movePercent * 65);
        } else {
            this.mesh.position.x = this.initialX - (movePercent * 400);

            let newPos = this.mesh.position.clone();
            newPos.add(this.hitboxOffset);
            this.hitbox.position.copy(newPos);
        }

    }



    let camera = null;
    let firstCameraPos = null
    let renderer = null;
    let ambientLight = null;
    let directionalLight = null;
    let sun = null;
    let flareLight = null;
    let controls = null;
    let oceanMaterial = null;
    let floatingObjects = null;
    let submarineMesh = null;
    let submarineItems = null;
    let itemClicked = null;
    let tvMesh = null;
    let tvButtonMesh = null;
    let raycaster = new Raycaster();
    let mouse = new Vector2();
    const scene = new Scene();
    let composer, outlinePass;

    function init() {

        //set page scroll to 0
        document.body.scrollTop = 0;
        document.documentElement.scrollTop = 0;


        scene.fog = new Fog(0x87CEEB, 50, 200);


        let width = window.innerWidth;
        let height = window.innerHeight;


        camera = new PerspectiveCamera(75, width/height, 0.1, 10000);
        

        firstCameraPos = new Vector3(0, 0, 30);

        camera.position.set(firstCameraPos.x, firstCameraPos.y, firstCameraPos.z);
        camera.rotation.x = 0.1;

        
        renderer = new WebGLRenderer({
            canvas: document.querySelector('#bg'),
            alpha: true,
        });

        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(width, height);
        renderer.setClearColor(0x87CEEB, 1);


        ambientLight = new AmbientLight(0xffffff, 0.5);
        scene.add(ambientLight);

        directionalLight = new DirectionalLight(0xffffff, 1);
        directionalLight.position.set(0, 50, 0);
        scene.add(directionalLight);

        const sunGeometry = new SphereGeometry(0.1, 32, 32);
        const sunMaterial = new MeshBasicMaterial({ color: 0xffff00 });
        sun = new Mesh(sunGeometry, sunMaterial);
        sun.position.set(-450, 50, -500);

        flareLight = new PointLight(0xffffff, 2, 500);
        flareLight.position.set(-450, 50, -500);

        flareLight.add(lensflare);
        scene.add(flareLight);


        floatingObjects = [];
        const numObjects = 10;
        for (let i = 0; i < numObjects; i++) {
            
            const turtleGeometry = new SphereGeometry(1, 8, 8);
            const turtleMaterial = new MeshStandardMaterial({ color: 0x2d5a27 });
            const turtle = new Mesh(turtleGeometry, turtleMaterial);

            turtle.position.set(
                MathUtils.randFloatSpread(100),
                MathUtils.randFloatSpread(40) - 25,
                MathUtils.randFloatSpread(100)
            );

            scene.add(turtle);

            floatingObjects.push({
                mesh: turtle,
                speed: MathUtils.randFloat(0.1, 0.3),
                initialX: turtle.position.x,
                initialZ: turtle.position.z,
                movementOffset: MathUtils.randFloat(0, Math.PI * 2)
            });

            const plasticGeometry = new BoxGeometry(1, 1, 1);
            const plasticMaterial = new MeshStandardMaterial({ 
                color: 0xff0000,
                roughness: 0.5,
                metalness: 0.5
            });
            const plastic = new Mesh(plasticGeometry, plasticMaterial);

            plastic.position.set(
                MathUtils.randFloatSpread(100),
                MathUtils.randFloatSpread(40) - 25,
                MathUtils.randFloatSpread(100)
            );

            scene.add(plastic);

            floatingObjects.push({
                mesh: plastic,
                speed: MathUtils.randFloat(0.1, 0.3),
                initialX: plastic.position.x,
                initialZ: plastic.position.z,
                movementOffset: MathUtils.randFloat(0, Math.PI * 2)
            });

        }

        
        const shipGeometry = new BoxGeometry(100, 20, 75);
        const shipMaterial = new MeshStandardMaterial({ 
            color: 0x8B4513,
            roughness: 0.8,
            metalness: 0.2,
            emissive: 0x8B4513,
            emissiveIntensity: 0.2,
            fog: false
        });
        const shipMesh = new Mesh(shipGeometry, shipMaterial);
        
        shipMesh.position.set(0, 5, -200);
        scene.add(shipMesh);


        floatingObjects.push({
            mesh: shipMesh,
            speed: 0.2,
            initialX: shipMesh.position.x,
            initialZ: shipMesh.position.z,
            initialShipX: shipMesh.position.x,
            initialShipZ: shipMesh.position.z,
            movementOffset: 0,
            isShip: true
        });


        const textGeometry = new TextGeometry(title, {
            font: textFont,
            size: 10,
            height: 1,
            curveSegments: 12,
            bevelEnabled: true,
            bevelThickness: 0.5,
            bevelSize: 0.5,
            bevelOffset: 0,
            bevelSegments: 5
        });
        const textMaterial = new MeshStandardMaterial({ color: 0xffffff, fog:false });
        const titleMesh = new Mesh(textGeometry, textMaterial);
        
        titleMesh.position.set(-30, 30, -150);
        scene.add(titleMesh);

        floatingObjects.push({
            mesh: titleMesh,
            speed: 0.2,
            initialX: titleMesh.position.x,
            initialZ: titleMesh.position.z,
            initialShipX: shipMesh.position.x,
            initialShipZ: shipMesh.position.z,
            movementOffset: 0,
            isShip: true
        });


        const backdropGeometry = new BoxGeometry(120, 40, 1);
        const backdropMaterial = new MeshStandardMaterial({ color: 0x000000, fog:false });
        const backdropMesh = new Mesh(backdropGeometry, backdropMaterial);
        
        backdropMesh.position.set(-30, 30, -200);
        scene.add(backdropMesh);
        
        floatingObjects.push({
            mesh: backdropMesh,
            speed: 0.2,
            initialX: backdropMesh.position.x,
            initialZ: backdropMesh.position.z,
            initialShipX: shipMesh.position.x,
            initialShipZ: shipMesh.position.z,
            movementOffset: 0,
            isShip: true
        });


        const seabedItems = [];
        const numSeabedItems = 20;
        for (let i = 0; i < numSeabedItems; i++) {

            const itemGeometry = new BoxGeometry(2, 2, 2);
            const itemMaterial = new MeshStandardMaterial({ 
                color: 0xff0000,
                roughness: 0.8,
                metalness: 0.2
            });
            const item = new Mesh(itemGeometry, itemMaterial);

            item.position.set(
                MathUtils.randFloatSpread(180),
                -oceanDepth + 2,
                MathUtils.randFloatSpread(180)
            );

            item.rotation.x = MathUtils.randFloat(0, Math.PI);
            item.rotation.y = MathUtils.randFloat(0, Math.PI);
            item.rotation.z = MathUtils.randFloat(0, Math.PI);

            scene.add(item);
            seabedItems.push(item);
        
        }


        controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.enabled = controlsEnabled;


        const oceanGeometry = new PlaneGeometry(1500, 1000, 50, 50);
        oceanMaterial = new ShaderMaterial({
            uniforms: {
                time: { value: 0 },
                color: { value: new Color(0x0077ff) },
                darkness: { value: 0 }
            },
            vertexShader: `
                uniform float time;
                varying vec2 vUv;
                varying float vHeight;
                void main() {
                    vUv = uv;
                    vec3 pos = position;

                    float wave1 = sin(pos.x * 0.05 + time) * 2.0;
                    float wave2 = cos(pos.y * 0.08 + time * 0.8) * 1.5;
                    float wave3 = sin((pos.x + pos.y) * 0.1 + time * 1.2) * 1.0;

                    float wave = wave1 + wave2 + wave3;
                    pos.z += wave;

                    vHeight = wave;

                    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
                }
            `,
            fragmentShader: `
                uniform vec3 color;
                uniform float darkness;
                varying vec2 vUv;
                varying float vHeight;
                void main() {
                    vec3 dayColor = vec3(0.0, 0.47, 1.0);
                    vec3 nightColor = vec3(0.0, 0.2, 0.4);

                    vec3 baseColor = mix(dayColor, nightColor, darkness);

                    vec3 darkColor = baseColor * 0.8;
                    vec3 lightColor = baseColor * 1.2;

                    vec3 finalColor = mix(darkColor, lightColor, (vHeight + 4.5) / 9.0);

                    finalColor *= 0.5 + (vHeight + 4.5) / 9.0;

                    gl_FragColor = vec4(finalColor, 1.0);
                }
            `,
            transparent: true
        });
        const ocean = new Mesh(oceanGeometry, oceanMaterial);
        
        ocean.rotation.x = -Math.PI / 2;
        ocean.position.y = 0;
        scene.add(ocean);


        const floorGeometry = new PlaneGeometry(1000, 1000);
        const floorMaterial = new MeshStandardMaterial({ 
            color: 0xb58c4e,
            roughness: 0.8,
            metalness: 0.2
        });
        const floor = new Mesh(floorGeometry, floorMaterial);
        
        floor.rotation.x = -Math.PI / 2;
        floor.position.y = -oceanDepth;
        scene.add(floor);


        textDepths.forEach((depth, index) => {

            if (textLabels[index] == "submarine") {

                submarineMesh = SkeletonUtils.clone(submarineScene);//group object

                
                submarineMesh.position.y = -oceanDepth * depth;
                submarineMesh.position.x = 50;
                scene.add(submarineMesh);

                floatingObjects.push({
                    mesh: submarineMesh,
                    speed: 0.2,
                    initialX: submarineMesh.position.x,
                    initialZ: submarineMesh.position.z,
                    depth: depth,
                    isSubmarine: true,
                    updatePos: updatePosition,
                });

                return;
            }


            const textGeometry = new TextGeometry(textLabels[index], {
                font: textFont,
                size: 5,
                height: 1,
                curveSegments: 12,
                bevelEnabled: true,
                bevelThickness: 0.5,
                bevelSize: 0.3,
                bevelSegments: 5,
            });
            const textMaterial = new MeshStandardMaterial({
                color: 0xffffff,
                fog: false,
            });
            const textMesh = new Mesh(textGeometry, textMaterial);

            textMesh.position.set(200, -oceanDepth * depth, -50);
            scene.add(textMesh);

            floatingObjects.push({
                mesh: textMesh,
                speed: MathUtils.randFloat(0.1, 0.3),
                initialX: textMesh.position.x,
                initialZ: textMesh.position.z,
                movementOffset: MathUtils.randFloat(0, Math.PI * 2),
                depth: depth,
                isFloatingText: true,
                isLink: Math.random() < 1,//random link idk bro testing
                link: "https://www.nationalgeographic.com/environment/article/what-is-geothermal-energy",
                updatePos: updatePosition,
                hitboxOffset: (() => {

                    let hitboxOffset = new Vector3(0, 0, 0);
                    textGeometry.computeBoundingBox();
                    textGeometry.boundingBox.getCenter(hitboxOffset);
                    
                    return hitboxOffset;

                })(),
                hitbox: (() => {

                    let size = new Vector3(0, 0, 0);
                    textGeometry.computeBoundingBox();
                    textGeometry.boundingBox.getSize(size);

                    const hitboxGeometry = new BoxGeometry(size.x, size.y, size.z);
                    const hitboxMaterial = new MeshBasicMaterial({ visible: true, opacity: 0.5, transparent: true });
                    const hitboxMesh = new Mesh(hitboxGeometry, hitboxMaterial);
                    
                    hitboxMesh.position.copy(textMesh.position);
                    scene.add(hitboxMesh);

                    return hitboxMesh;

                })(),
            });

        });



        // Set up postprocessing
        composer = new EffectComposer(renderer);
        const renderPass = new RenderPass(scene, camera);
        composer.addPass(renderPass);

        outlinePass = new OutlinePass(
            new Vector2(window.innerWidth, window.innerHeight),
            scene,
            camera
        );
        outlinePass.edgeStrength = 5;
        outlinePass.edgeGlow = 0.5;
        outlinePass.edgeThickness = 1.5;
        outlinePass.pulsePeriod = 1.5;
        outlinePass.visibleEdgeColor.set(0xf7f387);
        outlinePass.hiddenEdgeColor.set(0x000000);
        composer.addPass(outlinePass);

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();

            renderer.setSize(window.innerWidth, window.innerHeight);
            composer.setSize(window.innerWidth, window.innerHeight);
        }, false);


        submarineItems = submarineMesh.children.map(item => {
                if (item.name == "Bag" || item.name == "Disk" || item.name == "Bottle") 
                    if (item.name == "Disk")
                        return item.children[1];
                    else
                        return item;
                else
                    return null;
        }).filter(Boolean);


        //order of bag, bottle, disk
        submarineItems.sort((a, b) => {
            if (a.name < b.name) return -1;
            if (a.name > b.name) return 1;
            return 0;
        });


        //-1 = nothing clicked, 0 = bag, 1 = bottle, 2 = disk
        itemClicked = -1;

        const tvGeometry = new PlaneGeometry(0.43, 0.4);
        const tvMaterial = new ShaderMaterial({
            uniforms: {
                scroll: {value: 0},
                text: { value: tvTxtTexture },
                height: { value: tvTxtTexture.source.data.naturalHeight },
                screenHeight: { value: 1200 }
            },
            vertexShader: `
                varying vec2 vUv;
                void main() {
                    vUv = uv;
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
                }
            `,
            fragmentShader: `
                uniform sampler2D text;
                uniform float scroll;
                uniform float height;
                uniform float screenHeight;
                varying vec2 vUv;
                void main() {
                    vec2 nvUv = vec2(vUv.x, ((1.0-scroll)*height + vUv.y*screenHeight)/(height+screenHeight));
                    vec4 color = texture2D(text, nvUv);
                    if (nvUv.y > 1.0) {
                        gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);
                    } else {
                        gl_FragColor = vec4(color.xyz, 1.0);
                    }
                }
            `,
            transparent: true
        });

        tvMesh = new Mesh(tvGeometry, tvMaterial);
        scene.add(tvMesh);


        tvButtonMesh = submarineMesh.children.filter(item => item.name == "Button")[0];



        //TODO: make armature not frustum culled to not disapear
        // scene.frustumCulled = false;
        scene.traverse( function( object ) { object.frustumCulled = false; } );
        
        
        // scene.traverse( (object) => { 
        //     console.log(object)
        //     if (object.name == "Armature") object.frustumCulled = false; 
        // });

        console.log(scene);






    }

    let bottleMixer = null;
    let armatureMixer = null;
    function addFunctions() {
        window.addEventListener('keyup', (event) => {

            if (event.key === 'p') {

                controlsEnabled = !controlsEnabled;
                controls.enabled = controlsEnabled;

                if (!controlsEnabled) window.location.reload();
                
            }

        });

        window.addEventListener('mousemove', (event) => {

            mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
            mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        });

        window.addEventListener('mouseup', (event) => {

            raycaster.setFromCamera(mouse, camera);


            const items = floatingObjects.map(obj => (obj.isFloatingText && obj.isLink) ? obj.hitbox : null).filter(Boolean) 
            
            submarineItems.forEach(item => {
                items.push(item);
            });
            items.push(tvButtonMesh);
            
            
            const intersects = raycaster.intersectObjects(items);


            if (intersects.length > 0) {
                
                const intersectedItem = intersects[0].object;
                const linkedObject = floatingObjects.find(obj => obj.hitbox === intersectedItem);
                console.log(submarineMesh)
                if (intersectedItem.name == "Bottle") {

                    bottleMixer = new THREE.AnimationMixer(intersectedItem);
                    let armature = submarineMesh.children.filter(item => item.name == "Armature")[0];
                    console.log("armature")
                    console.log(armature)
                    console.log("intersecteditem")
                    console.log(intersectedItem)
                    armatureMixer = new THREE.AnimationMixer(armature);

                    // console.log("mixer")
                    // console.log(mixer)
                    
                    // console.log("submarineAnimations[1]")
                    // console.log(submarineAnimations[1])
                    
                    // let nclip = THREE.AnimationUtils.subclip( submarineAnimations[1], 'BottleAction', 0, 100, 30);
                    // let action = mixer.clipAction( nclip )
                    let action = bottleMixer.clipAction( submarineAnimations[1] )
                    
                    // action.setLoop(THREE.LoopOnce);
                    // action.clampWhenFinished = true;
                    action.play();
                    
                    let action2 = armatureMixer.clipAction( submarineAnimations[0] )
                    
                    console.log("mixer")
                    console.log(armatureMixer)
                    
                    console.log("submarineAnimations[0]")
                    console.log(submarineAnimations[0])
                    // action2.setLoop(THREE.LoopOnce);
                    // action2.clampWhenFinished = true;
                    action2.play();
                    // console.log("playing");



                    return;
                }


                
                if (linkedObject) {
                    window.location.href = linkedObject.link;
                } else if (itemClicked == -1) {
                    itemClicked = submarineItems.indexOf(intersectedItem);
                    tvAnimFinished = false;
                    tvAnimCurrentTime = 0;
                    prevTime = performance.now() * 0.001;

                    document.body.scrollTop = 0;
                    document.documentElement.scrollTop = 0;
                    
                } else if (itemClicked != -1) {
                    tvAnimFinished = false;
                    itemClicked = -1;
                    tvAnimCurrentTime = 0;
                    prevTime = performance.now() * 0.001;
                    
                    document.body.scrollTop = document.body.scrollHeight;
                    document.documentElement.scrollTop = document.documentElement.scrollHeight;
                }

            }

        });

        window.addEventListener( 'resize', () => {

            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();

            renderer.setSize( window.innerWidth, window.innerHeight );
        
        }, false);

        
    }
    
    let cameraSubStart = null;
    let cameraSubEnd = null;
    let cameraRotSubStart = null;
    let cameraRotSubEnd = null;
    let cameraSubEndOffset = new Vector3(0.1, 1.45, -2);
    function moveCamera() {

        const t = document.body.getBoundingClientRect().top;
        const mainHeight = document.querySelector("main").scrollHeight;
        const windowHeight = window.innerHeight;

        const scrollProgress = Math.abs(t) / (mainHeight - windowHeight);


        if (itemClicked == -1) {

            const sunPhase = Math.min(scrollProgress / sunPhasePercentage, 1);
    
            const startX = -450;
            const endX = 0;
            const sunX = startX + (endX - startX) * sunPhase;
    
            const baseHeight = 100;
            const heightVariation = 200;
            const easing = t => t * t * (3 - 2 * t);
            const easedPhase = easing(sunPhase);
            const sunY = baseHeight + Math.sin(easedPhase * Math.PI * 0.5) * heightVariation;
    
            flareLight.position.set(sunX, sunY, sun.position.z - 700);
    
            if (camera.position.y <= 0) {
                scene.remove(flareLight);
            } else {
                if (!scene.children.includes(flareLight)) {
                    scene.add(flareLight);
                }
            }
    
            const cameraOceanPhase = Math.min(1, Math.max(0, (scrollProgress - sunPhasePercentage) / (oceanPhasePercentage)));
            const cameraSubmarinePhase = Math.min(1, Math.max(0, (scrollProgress - oceanPhasePercentage - sunPhasePercentage) / (submarinePhasePercentage)));
            
            const maxDepth = oceanDepth + 25;
            
            
    
            phaseif: 
            if (cameraOceanPhase != 1) {
                camera.position.y = -cameraOceanPhase * maxDepth + 50;
                camera.rotation.x = -Math.PI / 8;
                cameraSubStart = null;
                cameraSubEnd = null;
            } else if (cameraOceanPhase == 1 && cameraSubmarinePhase != 0) {
    
                if (cameraSubStart == null) {
    
                    if (cameraSubEnd == null) {
                        cameraSubEnd = new Vector3(0, 0, 0);
                        break phaseif;
                    }
    
                    cameraSubStart = firstCameraPos.clone();
                    cameraSubStart.y = -maxDepth + 50;
    
                    cameraSubEnd =  submarineMesh.position.clone();
                    cameraRotSubStart = camera.rotation.clone();
                    cameraRotSubEnd = new Vector3(0, 0, 0);
    
                    cameraSubEnd.add(cameraSubEndOffset);
    
                }
    
    
                camera.position.x = MathUtils.lerp(cameraSubStart.x, cameraSubEnd.x, cameraSubmarinePhase);
                camera.position.y = MathUtils.lerp(cameraSubStart.y, cameraSubEnd.y, cameraSubmarinePhase);
                camera.position.z = MathUtils.lerp(cameraSubStart.z, cameraSubEnd.z, cameraSubmarinePhase);
    
                camera.rotation.x = MathUtils.lerp(cameraRotSubStart.x, cameraRotSubEnd.x, cameraSubmarinePhase);
                camera.rotation.y = MathUtils.lerp(cameraRotSubStart.y, cameraRotSubEnd.y, cameraSubmarinePhase);
                camera.rotation.z = MathUtils.lerp(cameraRotSubStart.z, cameraRotSubEnd.z, cameraSubmarinePhase);
            }
    
    
    
    
            const darkness = (darknessEnabled) ? Math.abs(Math.sin((sunPhase/2) * Math.PI - Math.PI/2)) : 0;
    
            scene.background = new Color(0x87CEEB).lerp(new Color(0x000033), darkness);
            ambientLight.intensity = 0.5 * (1 - darkness);
            directionalLight.intensity = 1 * (1 - darkness);
    
            scene.fog.density = 0.001 + (darkness * 0.002);
    
            oceanMaterial.uniforms.darkness.value = darkness;
    
            floatingObjects.forEach(obj => {
    
                if (obj.isShip) {
                    if (camera.position.y >= 0) {
                        const movement = Math.sin(scrollProgress * Math.PI * 2 + obj.movementOffset) * 100;
                        obj.mesh.position.x = obj.initialShipX + movement;
                        obj.mesh.position.z = obj.initialShipZ + movement;
                    }
                } else if (obj.isFloatingText || obj.isSubmarine) {
                    obj.updatePos(scrollProgress); 
                } else {
                    const movement = Math.sin(scrollProgress * Math.PI * 2 + obj.movementOffset) * 20;
                    obj.mesh.position.x = obj.initialX + movement;
                    obj.mesh.position.z = obj.initialZ + movement;
                }
    
            });


            let pos = submarineMesh.children.find(item => item.name == "Screen").position;

            tvMesh.position.set(pos.x, pos.y, pos.z+0.02);
            tvMesh.position.add(submarineMesh.position);
            
            tvMesh.material.uniforms.scroll.value = 2;

            
        } else {
            
            //item is clicked, display text on scroll
            tvMesh.material.uniforms.scroll.value = scrollProgress;

        }


    }
    
    let tvAnimFinished = true;
    let tvAnimTime = .5;
    let tvAnimCurrentTime = 0;
    let prevTime = 0;
    function animate() {
        requestAnimationFrame(animate);

        // document.body.onscroll = (itemClicked == -1) ? moveCamera : null;
        document.body.onscroll = moveCamera;


        if (loadedCount == MAX_LOADED) {
            init();
            moveCamera();
            loadedCount = -1;
        } else if (loadedCount == -1) {

            const time = performance.now() * 0.001;
            let delta = time - prevTime; //seconds


            if (bottleMixer) bottleMixer.update(delta);
            if (armatureMixer) armatureMixer.update(delta);


            oceanMaterial.uniforms.time.value = time;

            floatingObjects.forEach(obj => {
                if (obj.isShip) {
                    const time = performance.now() * 0.001;
                    const waveRotation = Math.sin(time * 2 + obj.movementOffset) * 0.1;
                    obj.mesh.rotation.z = waveRotation;
                    obj.mesh.rotation.x = Math.sin(time * 1.5 + obj.movementOffset) * 0.05;
                }
            });

            raycaster.setFromCamera(mouse, camera);

           
           
            const items = floatingObjects.map(obj => (obj.isFloatingText && obj.isLink) ? obj.hitbox : null).filter(Boolean);
            submarineItems.forEach(item => {
                items.push(item);
            });
            // const items = [];
            // submarineMesh.traverse((item) => {if (item.isMesh) items.push(item)});
            if (itemClicked != -1) items.push(tvButtonMesh);

            const intersects = raycaster.intersectObjects(items);



            if (intersects.length > 0) {
                const intersectedItem = intersects[0].object;
                composer.passes[1].selectedObjects = [intersectedItem];

                composer.passes[1].visibleEdgeColor.set(0xffffff);
                composer.passes[1].pulsePeriod = 0;
            } else if (itemClicked != -1) {
                composer.passes[1].selectedObjects = [tvButtonMesh];

                composer.passes[1].visibleEdgeColor.set(0x0062ff);
                composer.passes[1].pulsePeriod = 1.5;
            } else {
                composer.passes[1].selectedObjects = items;

                composer.passes[1].visibleEdgeColor.set(0x0062ff);
                composer.passes[1].pulsePeriod = 1.5;
            }


            if (!tvAnimFinished) {

                let t = Math.min(1, tvAnimCurrentTime / tvAnimTime);
                
                let initial = cameraSubEnd.clone();
                let final = (new Vector3(-.1, -.1, -1.1)).add(initial).clone();
                
                tvAnimCurrentTime += delta;
                
                camera.position.x = MathUtils.lerp(initial.x, final.x, (itemClicked != -1) ? t : 1-t);
                camera.position.y = MathUtils.lerp(initial.y, final.y, (itemClicked != -1) ? t : 1-t);
                camera.position.z = MathUtils.lerp(initial.z, final.z, (itemClicked != -1) ? t : 1-t);
                
                tvAnimFinished = (itemClicked != -1) ? t == 1 : 1-t == 0;
            } 












            composer.render();
            prevTime = time;
        }

    }


    onMount( () => {
        loadStuff();
        addFunctions();
        animate();
    });

</script>



<canvas id="bg"></canvas>

<main>
    {#if !controlsEnabled}
        <div id="scroll-window" style='height:{scrollWindowHeight}px'></div>
    {/if}
</main>

<style>
    canvas {
        position: fixed;
        top: 0;
        left: 0;
    }

    main {
        width: 100%;
        z-index: 1;
        position: absolute;
        background-color: transparent;
    }
</style>