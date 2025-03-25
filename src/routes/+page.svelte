<!-- https://www.eia.gov/energyexplained/geothermal/images/drysteam.gif -->
<script>
  import Header from '../sections/header.svelte'
  import Map from '../sections/map.svelte'
  import Coursell from '../sections/carousel.svelte'

    import * as THREE from 'three';
    
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
    import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader'
    import { Lensflare, LensflareElement } from 'three/examples/jsm/objects/Lensflare.js';
    
    import { onMount } from 'svelte';
    import Carousel from '../sections/carousel.svelte';
    onMount(() => {
    
        // IMPORTS
        // import './style.css';
        
        var clock;
        // SETUP
        const scene = new THREE.Scene();
        scene.fog = new THREE.Fog(0x87CEEB, 50, 200); // Add fog for depth perception
        
        // Add darkness toggle
        const enableDarkness = false; // Toggle this to enable/disable darkness effect
        
        let width = window.innerWidth;
        let height = window.innerHeight;
    
        const camera = new THREE.PerspectiveCamera(75, width/height, 0.1, 10000);
        
        const renderer = new THREE.WebGLRenderer({
          canvas: document.querySelector('#bg'),
          alpha: true
        });
        
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(width, height);
        renderer.setClearColor(0x87CEEB, 1);
    
        // Set initial camera position and rotation
        camera.position.set(0, 0, 30);
        camera.rotation.x = 0.1; // Set a fixed downward angle
        
        // SCENE OBJECTS
        // const geometry = new THREE.TorusGeometry( 10, 3, 16, 100 );
        // const material = new THREE.MeshStandardMaterial( { color: 0xFF6347 } );
        // const torus = new THREE.Mesh( geometry, material );
        
        // scene.add(torus); 
        
        // const pointLight = new THREE.PointLight(0xffffff, 1050, 50);
        // pointLight.position.set(0, 0, -50);
        
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
        // scene.add(pointLight, ambientLight);
        scene.add(ambientLight);
    
        const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
        directionalLight.position.set(0, 50, 0);
        scene.add(directionalLight);
    
        
        // const targetObject = new THREE.Object3D();
        // scene.add(targetObject);
    
        // directionalLight.target = targetObject;
    
        
        // const lightHelper = new THREE.PointLightHelper(pointLight);
        // const gridHelper = new THREE.GridHelper(200, 50);
        // scene.add(lightHelper, gridHelper);
        
        const sunGeometry = new THREE.SphereGeometry(0.1, 32, 32);
        const sunMaterial = new THREE.MeshBasicMaterial({ color: 0xffff00 });
        const sun = new THREE.Mesh(sunGeometry, sunMaterial);
        sun.position.set(-450, 50, -500);
        scene.add(sun);

        const flare_light = new THREE.PointLight(0xffffff, 2, 500);
        const textureLoader = new THREE.TextureLoader();
        const lensflare = new Lensflare();

        const flareTextures = [
          { texture: "/img/solar/mainFlare.png", size: 450, strength: 0.2, color: new THREE.Color(0x856319) },
          { texture: "/img/solar/orangeFlareM.png", size: 128, strength: 0.3, color: new THREE.Color(0x856319) },
          { texture: "/img/solar/orangeFlareS.png", size: 64, strength: 0.4, color: new THREE.Color(0x856319) }
        ];

        flareTextures.forEach(({ texture, size, strength, color }) => {
          lensflare.addElement(new LensflareElement(
            textureLoader.load(texture),
            size,
            strength,
            color
          ));
        });

        flare_light.add(lensflare);
        scene.add(flare_light);
        
        // Create floating objects
        const floatingObjects = [];
        const numObjects = 10;
        
        for (let i = 0; i < numObjects; i++) {
          // Create turtles
          const turtleGeometry = new THREE.SphereGeometry(1, 8, 8);
          const turtleMaterial = new THREE.MeshStandardMaterial({ color: 0x2d5a27 });
          const turtle = new THREE.Mesh(turtleGeometry, turtleMaterial);
          
          // Random position
          turtle.position.set(
            THREE.MathUtils.randFloatSpread(100),
            THREE.MathUtils.randFloatSpread(40) - 25,
            THREE.MathUtils.randFloatSpread(100)
          );
          
          // Add to scene and array
          scene.add(turtle);
          floatingObjects.push({
            mesh: turtle,
            speed: THREE.MathUtils.randFloat(0.1, 0.3),
            initialX: turtle.position.x,
            initialZ: turtle.position.z,
            movementOffset: THREE.MathUtils.randFloat(0, Math.PI * 2) // Random offset for independent movement
          });
          
          // Create plastic objects
          const plasticGeometry = new THREE.BoxGeometry(1, 1, 1);
          const plasticMaterial = new THREE.MeshStandardMaterial({ 
            color: 0xff0000,
            roughness: 0.5,
            metalness: 0.5
          });
          const plastic = new THREE.Mesh(plasticGeometry, plasticMaterial);
          
          // Random position
          plastic.position.set(
            THREE.MathUtils.randFloatSpread(100),
            THREE.MathUtils.randFloatSpread(40) - 25,
            THREE.MathUtils.randFloatSpread(100)
          );
          
          // Add to scene and array
          scene.add(plastic);
          floatingObjects.push({
            mesh: plastic,
            speed: THREE.MathUtils.randFloat(0.1, 0.3),
            initialX: plastic.position.x,
            initialZ: plastic.position.z,
            movementOffset: THREE.MathUtils.randFloat(0, Math.PI * 2) // Random offset for independent movement
          });
        }

        // Create scattered items on seabed
        const seabedItems = [];
        const numSeabedItems = 20;
        
        for (let i = 0; i < numSeabedItems; i++) {
          const itemGeometry = new THREE.BoxGeometry(2, 2, 2);
          const itemMaterial = new THREE.MeshStandardMaterial({ 
            color: 0xff0000,
            roughness: 0.8,
            metalness: 0.2
          });
          const item = new THREE.Mesh(itemGeometry, itemMaterial);
          
          // Random position on seabed
          item.position.set(
            THREE.MathUtils.randFloatSpread(180),
            -48, // Just above the seabed
            THREE.MathUtils.randFloatSpread(180)
          );
          
          // Random rotation
          item.rotation.x = THREE.MathUtils.randFloat(0, Math.PI);
          item.rotation.y = THREE.MathUtils.randFloat(0, Math.PI);
          item.rotation.z = THREE.MathUtils.randFloat(0, Math.PI);
          
          scene.add(item);
          seabedItems.push(item);
        }
        
        // Controls
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.enabled = false; // Disable orbit controls for straight down movement
        
        // Ocean surface
        const oceanGeometry = new THREE.PlaneGeometry(1500, 1000, 50, 50);
        const oceanMaterial = new THREE.ShaderMaterial({
          uniforms: {
            time: { value: 0 },
            color: { value: new THREE.Color(0x0077ff) }
          },
          vertexShader: `
            uniform float time;
            varying vec2 vUv;
            varying float vHeight;
            void main() {
              vUv = uv;
              vec3 pos = position;
              
              // Multiple wave frequencies for more complex movement
              float wave1 = sin(pos.x * 0.05 + time) * 2.0;
              float wave2 = cos(pos.y * 0.08 + time * 0.8) * 1.5;
              float wave3 = sin((pos.x + pos.y) * 0.1 + time * 1.2) * 1.0;
              
              // Combine waves with different amplitudes
              float wave = wave1 + wave2 + wave3;
              pos.z += wave;
              
              // Pass height to fragment shader for coloring
              vHeight = wave;
              
              gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
          `,
          fragmentShader: `
            uniform vec3 color;
            varying vec2 vUv;
            varying float vHeight;
            void main() {
              // Base color
              vec3 baseColor = vec3(0.0, 0.47, 1.0); // Blue
              
              // Add darker color for wave troughs and lighter for peaks
              vec3 darkColor = vec3(0.0, 0.3, 0.8);  // Darker blue
              vec3 lightColor = vec3(0.2, 0.6, 1.0); // Lighter blue
              
              // Mix colors based on wave height
              vec3 finalColor = mix(darkColor, lightColor, (vHeight + 4.5) / 9.0);
              
              // Add slight transparency variation based on height
              float alpha = 0.8 + (vHeight + 4.5) / 18.0;
              
              gl_FragColor = vec4(finalColor, alpha);
            }
          `,
          transparent: true
        });
        const ocean = new THREE.Mesh(oceanGeometry, oceanMaterial);
        ocean.rotation.x = -Math.PI / 2;
        ocean.position.y = 0;
        scene.add(ocean);

        // Sea floor
        const floorGeometry = new THREE.PlaneGeometry(1000, 1000);
        const floorMaterial = new THREE.MeshStandardMaterial({ 
          color: 0xb58c4e, // Sandy color
          roughness: 0.8,
          metalness: 0.2
        });
        const floor = new THREE.Mesh(floorGeometry, floorMaterial);
        floor.rotation.x = -Math.PI / 2;
        floor.position.y = -50;
        scene.add(floor);

        // Plastic trash
        const trashGeometry = new THREE.BoxGeometry(2, 2, 2);
        const trashMaterial = new THREE.MeshStandardMaterial({ 
          color: 0xff0000,
          roughness: 0.5,
          metalness: 0.5
        });
        const trash = new THREE.Mesh(trashGeometry, trashMaterial);
        trash.position.set(0, -45, 0);
        scene.add(trash);
        
        // ON SCROLL 
        function moveCamera() {
            const t = document.body.getBoundingClientRect().top;
            const mainHeight = document.querySelector("main").scrollHeight;
            const windowHeight = window.innerHeight;
            
            // Calculate scroll progress (0 to 1)
            const scrollProgress = Math.abs(t) / (mainHeight - windowHeight);
            
            // Sun movement phase (first 30% of scroll)
            const sunPhasePercentage = 0.5;
            const sunPhase = Math.min(scrollProgress / sunPhasePercentage, 1);
            
            // Sun movement from left to right
            const startX = -450;
            const endX = 450;
            const sunX = startX + (endX - startX) * sunPhase;
            
            // Sun height movement (up and down)
            const baseHeight = 100;
            const heightVariation = 75;
            const sunY = baseHeight + Math.sin(sunPhase * Math.PI) * heightVariation;
            
            sun.position.set(sunX, sunY, sun.position.z);
            flare_light.position.set(sunX, sunY, sun.position.z + 10);
            
            //if camera position is below the sea level then hide the sun, and readd it when it goes back above
            if (sunPhase == 1) {
                scene.remove(sun);
                scene.remove(flare_light);
            } else {
                if (!scene.children.includes(sun)) {
                    scene.add(sun);
                    scene.add(flare_light);
                }
            }

            // Camera movement phase (after sun movement, last 70% of scroll)
            const cameraPhase = Math.max(0, (scrollProgress - sunPhasePercentage) / (1 - sunPhasePercentage));
            const maxDepth = 90; // Maximum depth (just above sea floor)
            camera.position.y = -cameraPhase * maxDepth + 50;
            camera.rotation.x = -Math.PI / 8;
            
            // Update lighting and background based on sun position
            const darkness = enableDarkness ? Math.abs(Math.sin((sunPhase) * Math.PI - Math.PI/2)) : 0;
            
            scene.background = new THREE.Color(0x87CEEB).lerp(new THREE.Color(0x000033), darkness);
            ambientLight.intensity = 0.5 * (1 - darkness);
            directionalLight.intensity = 1 * (1 - darkness);
            
            // Update fog based on sun position
            scene.fog.density = 0.001 + (darkness * 0.002);

            // Move floating objects independently based on scroll
            floatingObjects.forEach(obj => {
              const movement = Math.sin(scrollProgress * Math.PI * 2 + obj.movementOffset) * 20;
              obj.mesh.position.x = obj.initialX + movement;
              obj.mesh.position.z = obj.initialZ + movement;
            });
        }
        
        document.body.onscroll = moveCamera;
        moveCamera();
        
        // ON FRAME
        function animate() {
            requestAnimationFrame(animate);
            
            const time = performance.now() * 0.001;
            oceanMaterial.uniforms.time.value = time;
            
            // Remove rotation of floating objects
            
            renderer.render(scene, camera);
        }
        animate();


        
        window.addEventListener( 'resize', onWindowResize, false );
        
        
        
        function onWindowResize() {
        
          camera.aspect = window.innerWidth / window.innerHeight;
          camera.updateProjectionMatrix();
        
          renderer.setSize( window.innerWidth, window.innerHeight );
        
        }
    
        var acc = document.getElementsByClassName("accordion");
  
        acc[0].addEventListener("click", function() {
          this.classList.toggle("active");
          var text = this.nextElementSibling;
          if (text.style.maxHeight) {
          text.style.maxHeight = null;
          } else {
          text.style.maxHeight = text.scrollHeight + "px";
          } 
        });
        
        // });

    });
    
    </script>


<canvas id="bg"></canvas>

<main>
    <!-- <div class="landing-title-container">
        <p class="title">
            <b>Renewable Energy</b>
        </p>
    
        <p class="landing-desc">
            &emsp; &emsp; Welcome to our website, where we'll dive deep into the world of renewable energy in the United States! As you may know, the US is one of the largest energy consumers in the world, having a large energy consumption per person. In America, the average person uses about 85.87 Megawatt Hours of energy (MWh). Petroleums and natural gases are still by far the most consumed energy sources in America, however, renewable energy sources are becoming more and more widespread. In this website, we aim to educate you about some of the alternative energy sources that America has started to use, including hydroelectric, geothermal, and solar power, and their future in the U.S.!
        </p>
    </div>
    <div class="landing-container">
        
        <Carousel contentArr={[
            {
                img: "/img/geo/geoTitle.png", 
                colorPrefix: "geo",
                title: "Geothermal Energy",
                content: "Hydroelectric energy is one of the widely used renewable sources of energy. It harnesses the power of water by converting kinetic energy to electric energy. This energy can then be stored and used on demand, emitting relatively low greenhouse gases compared to fossil fuels. Sadly, constructing these massive hydroelectric dams can have large environmental and social impacts.",
                link: "./geothermal"
            },
            {
                img: "/img/hydro/hydroTitle.jpg", 
                colorPrefix: "hydro",
                title: "Hydroelectric Energy",
                content: "Geothermal energy is a unique and complex renewable energy source extracted from the Earth's crust. It uses an interesting process to turn heat from underground to energy. Geothermal energy is a reliable resource, providing a constant source of heat and electricity. However, the depth at which the heat sources are located poses a significant challenge in harnessing it. ",
                link: "./hydro"
            },
            {
                img: "/img/solar/solarTitle.jpg", 
                colorPrefix: "solar",
                title: "Solar Energy",
                content: "Solar energy is a renewable energy source that utilizes the sun's power. It's created by harnessing the photons from the sun by using large solar farms. The energy captured by the solar farms is then turned into electrical energy through the photovoltaic effect. Since the total amount of energy the earth receives from the sun is vastly more than our energy demands, solar energy has the potential to satisfy all future energy needs.",
                link: "./solar"
            },
        ]}/>
    </div>
    <div class="landing-map-container">
        <p class="map-text">Below is a map that, when hovered, shows the details of the energy production and the number of power plants each state has for a given renewable energy source. You can also change which statistic the heatmap shows by clicking on their respective icons.</p>
        <Map/>
    
        <button class="accordion"><p class="accordion-title">Citations</p> <p class="accordion-tip">Click to reveal!</p></button>
        <div class="accordion-text">
            <p>
                EIA. (n.d.). Frequently Asked Questions (FAQs) - U.S. Energy Information Administration. Frequently Asked Questions (FAQs) - U.S. Energy Information Administration (EIA). Retrieved April 15, 2023, from <a href="https://www.eia.gov/tools/faqs/faq.php?id=85&t=1">https://www.eia.gov/tools/faqs/faq.php?id=85&t=1</a>
                <br> 
                <br> 
                U.S. Energy Information Administration. (2022, October 27). Annual Energy Outlook 2022. Presentation to Electricity Advisory Committee. <a href="https://www.energy.gov/sites/default/files/2022-11/05%20October%2027%20-%20EIA%20Annual%20Energy%20Outlook%202022.pdf">https://www.energy.gov/sites/default/files/2022-11/05%20October%2027%20-%20EIA%20Annual%20Energy%20Outlook%202022.pdf</a>
                <br> 
                <br> 
            </p>
        </div>
    </div> -->
    <div id="lorem">
        Lorem ipsum dolor sit, amet consectetur adipisicing elit. Consequuntur doloribus, neque officiis, exercitationem impedit non tempore incidunt praesentium quisquam qui iusto. Molestias blanditiis aperiam, commodi quas a dignissimos sapiente non veniam ad laudantium nemo, aut doloremque quod? Non totam ratione modi facere sint vel impedit nihil laboriosam cupiditate eius minus adipisci itaque quaerat ad, repudiandae fuga magnam voluptates accusamus. Sunt autem reiciendis blanditiis laborum maxime officia vel, dolor, temporibus nihil vero omnis laboriosam expedita maiores voluptatum doloribus eos perferendis mollitia ex tempore totam nostrum inventore similique. Minus, fugiat rerum hic sit, nam harum incidunt aut natus fuga officiis error magni!
    </div>
</main>
 
<style>

    #lorem {
        font-size: 14rem;
        text-align: center;
        color: rgba(255, 255, 255, 0);
    }

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

    .landing-container {
        position: relative;
        width: auto;
        color: black;
        background-color: rgba(255, 255, 255, 0.8);
        margin: 5rem 6rem 0 6rem;
        padding: 2rem;
        border-radius: 1rem;
    }

    .landing-title-container {
        width: 120rem;
        max-width: 90vw;
        height: 80vh;
        background-color: rgba(255, 255, 255, 0.8);
        padding: 5vh 2rem;
        margin: 5vh auto;
        border-radius: 1rem;
    }

    .landing-map-container {
        width: 120rem;
        max-width: 90vw;
        background-color: rgba(255, 255, 255, 0.8);
        padding: 5vh 2rem;
        margin: 5vh auto;
        border-radius: 1rem;
    }

    .title {
        margin: 0;
        text-align: center;
        font-size: 14rem;
    }

    .landing-desc {
        font-size: 3rem;
        margin: 4rem 10rem;
        text-align: center;
    }

    .landing-map-container .map-text {
        font-size: 3rem;
        text-align: center;

        margin: 0 10rem 2rem 10rem;
    }

    .map-text {
        padding: 2rem;
        background-color: #ccc;
        border-radius: 1rem;
    }
</style>