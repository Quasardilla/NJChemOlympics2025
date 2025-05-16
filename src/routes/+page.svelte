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

    const oceanDepth = 300;
    const sunPhasePercentage = 0.2;
    const oceanPhasePercentage = 0.5;
    const submarinePhasePercentage = 0.3;

    const scrollWindowHeight = 5_000;


    let loadedCount = 0;
    const MAX_LOADED = 9; // 3 solar flare pngs, 1 font, 1 submarine glb, 4 research images (bag, bottle, disk, human)

    let submarineScene = null;
    let submarineAnimations = null;
    let lensflare = null;
    let textFont = null;
    let bagResearchTexture = null;
    let bottleResearchTexture = null;
    let diskResearchTexture = null;
    let humanResearchTexture = null;
    function loadStuff() {
        
        const gltfLoader = new GLTFLoader();
        const fontLoader = new FontLoader();
        const textureLoader = new TextureLoader();

        lensflare = new Lensflare();
        const flareTextures = [
            { texture: "/img/flares/mainFlare.png", size: 450, strength: 0.2, color: new Color(0x856319) },
            { texture: "/img/flares/orangeFlareM.png", size: 128, strength: 0.3, color: new Color(0x856319) },
            { texture: "/img/flares/orangeFlareS.png", size: 64, strength: 0.4, color: new Color(0x856319) }
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



            loadedCount++;
        });
        
        textureLoader.load('/img/research/Bag.png', (texture) => {
            bagResearchTexture = texture;
            loadedCount++;
        });
        textureLoader.load('/img/research/Bottle.png', (texture) => {
            bottleResearchTexture = texture;
            loadedCount++;
        });
        textureLoader.load('/img/research/Disk.png', (texture) => {
            diskResearchTexture = texture;
            loadedCount++;
        });
        textureLoader.load('/img/research/human.png', (texture) => {
            humanResearchTexture = texture;
            loadedCount++;
        });





    }




    const title = "Microplastics";
    const textDepths = [0.25, 0.5, 0.75, 0.90];
    const textLabels = ['Microplastics are small plastic \nparticles that come from the \ndegradation of plastics.', 
                        'Primary microplastics are plastic \nparticles designed to be very \nsmall so that they can carry out \ntheir intended function.', 
                        'Secondary microplastics are large \nplastic materials that could be \nused in packaging that get ground \ndown over time into microplastics.',
                        'submarine'];



    function updatePosition(scroll) {
    
        const padding = 0.2;
        const calcScroll = Math.min(1, Math.max(0, (scroll-sunPhasePercentage) / (oceanPhasePercentage)));
        
        if (calcScroll < this.depth - padding || calcScroll > this.depth + padding)
            return;
        
        const movePercent = (calcScroll - (this.depth - padding)) / ((this.depth + padding) - (this.depth - padding));
        
        if (this.isSubmarine) {
            this.mesh.position.x = this.initialX - (movePercent * 65);
        } else {

            let funcMovePercent = Math.log((1.0/movePercent)-1.0)/-7.0 + 0.5;
            funcMovePercent = (funcMovePercent>1) ? 1 : ((funcMovePercent < 0) ? 0 : funcMovePercent);

            this.mesh.position.x = this.initialX - (funcMovePercent * 400);

            let newPos = this.mesh.position.clone();
            newPos.add(this.hitboxOffset);
            this.hitbox.position.copy(newPos);
        }

    }

    function setScrollTop() {
        document.body.scrollTop = 0;
        document.documentElement.scrollTop = 0;
    }

    function setScrollBottom() {
        document.body.scrollTop = document.body.scrollHeight;
        document.documentElement.scrollTop = document.documentElement.scrollHeight;
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
    let humanShown = false;
    let tvMesh = null;
    let tvButtonMesh = null;
    let humanButtonMesh = null;
    let humanPlaneMesh = null;
    let raycaster = new Raycaster();
    let mouse = new Vector2();
    const scene = new Scene();
    let composer, outlinePass;

    function init() {

        setScrollTop()


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

        
        const textGeometry = new TextGeometry(title, {
            font: textFont,
            size: 20,
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
        
        titleMesh.position.set(-75, 30, -150);
        scene.add(titleMesh);


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

                
                submarineMesh.position.y = -oceanDepth + 20;
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
                // bevelEnabled: true,
                // bevelThickness: 0.5,
                // bevelSize: 0.3,
                // bevelSegments: 5,
            });
            const textMaterial = new MeshStandardMaterial({
                color: 0x000000,
                fog: false,
            });
            const textMesh = new Mesh(textGeometry, textMaterial);

            textMesh.position.set(200, -oceanDepth * depth, -50);
            textMesh.scale.set(1, 1, 0.5)
            scene.add(textMesh);

            floatingObjects.push({
                mesh: textMesh,
                speed: MathUtils.randFloat(0.1, 0.3),
                initialX: textMesh.position.x,
                initialZ: textMesh.position.z,
                movementOffset: MathUtils.randFloat(0, Math.PI * 2),
                depth: depth,
                isFloatingText: true,
                isLink: Math.random() < 0,//random link idk bro testing
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
                    // scene.add(hitboxMesh);

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
        outlinePass.edgeThickness = 3.5;
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
                text: { value: humanResearchTexture },
                height: { value: humanResearchTexture.source.data.naturalHeight },
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

        const planeMaterial = new ShaderMaterial({
            uniforms: {
                scroll: {value: 0},
                text: { value: humanResearchTexture },
                height: { value: humanResearchTexture.source.data.naturalHeight },
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
                    vec2 nvUv = vec2(vUv.y, ((1.0-scroll)*height + vUv.x*screenHeight)/(height+screenHeight));
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




        submarineMesh.children.forEach((obj) => {if (obj.name=="Plane001") {humanPlaneMesh = obj}})

        humanPlaneMesh.material=planeMaterial;
        humanPlaneMesh.rotation.y+=Math.PI;//


        tvButtonMesh = submarineMesh.children.filter(item => item.name == "Button")[0];
        humanButtonMesh = submarineMesh.children.filter(item => item.name == "Button2")[0];



        submarineMesh.children.forEach((obj) => {if (obj.name=="Armature001" || obj.name=="Plane001" || obj.name=="Plastic" || obj.name=="Water" || obj.name=="MaterialSphere") obj.visible=false;})


        console.log(submarineMesh)



    }

    let bottleMixer = null;
    let armatureMixer = null;
    let plasticMixer = null;
    let waterMixer = null;
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
            if (itemClicked!=-1) items.push(tvButtonMesh);
            else items.push(humanButtonMesh);
            
            
            const intersects = raycaster.intersectObjects(items);


            if (intersects.length > 0) {
                
                const intersectedItem = intersects[0].object;
                const linkedObject = floatingObjects.find(obj => obj.hitbox === intersectedItem);


                
                if (linkedObject) {
                    window.location.href = linkedObject.link;
                } else if (itemClicked == -1) {

                    if (intersectedItem.name == "Button2") {
                        
                        humanAnimFinished = false;
                        humanShown = true;

                        humanAnimCurrentTime = 0;
                        prevTime = performance.now() * 0.001;

                        setScrollTop()

                        
                        let armature = submarineMesh.children.filter(item => item.name == "Armature")[0];
                        let bottle = submarineMesh.children.filter(item => item.name == "Bottle")[0]
                        let plastic = submarineMesh.children.filter(item => item.name == "Plastic")[0]
                        let water = submarineMesh.children.filter(item => item.name == "Water")[0]
                        armatureMixer = new THREE.AnimationMixer(armature);
                        bottleMixer = new THREE.AnimationMixer(bottle);
                        plasticMixer = new THREE.AnimationMixer(plastic);
                        waterMixer = new THREE.AnimationMixer(water);

                        armatureMixer.addEventListener("finished", () => {
                            submarineMesh.children.forEach((obj) => {
                                if (obj.name=="Armature001" || obj.name=="Plastic" || obj.name=="Water" || obj.name=="Plane001") obj.visible=true;
                                if (obj.name=="Armature") obj.visible=false;
                            })
                            armatureMixer = null;

                            let action3 = plasticMixer.clipAction( submarineAnimations[3] )
                            action3.setLoop(THREE.LoopOnce);
                            action3.clampWhenFinished = true;
                            action3.play();
                            
                            let action4 = waterMixer.clipAction( submarineAnimations[4] )
                            action4.setLoop(THREE.LoopOnce);
                            action4.clampWhenFinished = true;
                            action4.play();


                        })

                        let action0 = armatureMixer.clipAction( submarineAnimations[0] )
                        action0.setLoop(THREE.LoopOnce);
                        action0.clampWhenFinished = true;
                        action0.play();
                        
                        console.log(submarineAnimations)
                        let action1 = bottleMixer.clipAction( submarineAnimations[2] )
                        action1.setLoop(THREE.LoopOnce);
                        action1.clampWhenFinished = true;
                        action1.play();

                        
                        return;

                    }

                    itemClicked = submarineItems.indexOf(intersectedItem);

                    //-1 = nothing clicked, 0 = bag, 1 = bottle, 2 = disk
                    if (itemClicked==0) {
                        tvMesh.material.uniforms.text.value = bagResearchTexture;
                        tvMesh.material.uniforms.height.value = bagResearchTexture.source.data.naturalHeight;
                    } else if (itemClicked==1) {
                        tvMesh.material.uniforms.text.value = bottleResearchTexture;
                        tvMesh.material.uniforms.height.value = bottleResearchTexture.source.data.naturalHeight;
                    } else if (itemClicked==2) {
                        tvMesh.material.uniforms.text.value = diskResearchTexture;
                        tvMesh.material.uniforms.height.value = diskResearchTexture.source.data.naturalHeight;
                    }


                    tvAnimFinished = false;
                    tvAnimCurrentTime = 0;
                    prevTime = performance.now() * 0.001;

                    setScrollTop()
                    
                } else if (itemClicked != -1) {
                    tvAnimFinished = false;
                    itemClicked = -1;
                    tvAnimCurrentTime = 0;
                    prevTime = performance.now() * 0.001;
                    

                    setScrollBottom()

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


        if (itemClicked == -1 && !humanShown) {

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

            
        } else if (itemClicked != -1 && !humanShown) {
            
            //item is clicked, display text on scroll
            tvMesh.material.uniforms.scroll.value = scrollProgress;
            
        } else if (itemClicked == -1 && humanShown) {
            humanPlaneMesh.material.uniforms.scroll.value = scrollProgress;
        }


    }
    
    let tvAnimFinished = true;
    let tvAnimTime = .5;
    let tvAnimCurrentTime = 0;
    let humanAnimFinished = true;
    let humanAnimTime = 1.5;
    let humanAnimCurrentTime = 0;
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
            if (plasticMixer) plasticMixer.update(delta);
            if (waterMixer) waterMixer.update(delta);


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
            if (!humanShown){
                submarineItems.forEach(item => {
                    items.push(item);
                });
            }
            // const items = [];
            // submarineMesh.traverse((item) => {if (item.isMesh) items.push(item)});
            if (itemClicked != -1) items.push(tvButtonMesh);
            else items.push(humanButtonMesh);

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


            if (!humanAnimFinished) {

                let t = Math.min(1, humanAnimCurrentTime / humanAnimTime);
                
                let initial = cameraSubEnd.clone();
                let final = (new Vector3(-.3, -.3, -.6)).add(initial).clone();
                
                let initialRot = cameraRotSubEnd.clone();
                let finalRot = (new Vector3(0, Math.PI/4, 0)).add(initialRot).clone();
                
                humanAnimCurrentTime += delta;
                
                camera.position.x = MathUtils.lerp(initial.x, final.x, (humanShown) ? t : 1-t);
                camera.position.y = MathUtils.lerp(initial.y, final.y, (humanShown) ? t : 1-t);
                camera.position.z = MathUtils.lerp(initial.z, final.z, (humanShown) ? t : 1-t);
                
                camera.rotation.x = MathUtils.lerp(initialRot.x, finalRot.x, (humanShown) ? t : 1-t);
                camera.rotation.y = MathUtils.lerp(initialRot.y, finalRot.y, (humanShown) ? t : 1-t);
                camera.rotation.z = MathUtils.lerp(initialRot.z, finalRot.z, (humanShown) ? t : 1-t);



                humanAnimFinished = (humanShown) ? t == 1 : 1-t == 0;
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