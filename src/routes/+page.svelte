<script lang="ts">
	import * as THREE from 'three/webgpu';
	import {
		positionLocal,
		Fn,
		If,
		Loop,
		Break,
		float,
		vec2,
		vec3,
		vec4,
		min,
		max,
		length,
		normalize,
		cross,
		dot,
		time,
		sin,
		cos,
		abs,
		negate,
		uniform,
		uniformArray,
		pow,
		reflect,
		refract,
		clamp,
		mod,
		exp,
		color,
		mix,
		int,
		smoothstep,
		round,
		sqrt,
		mx_noise_float,
		mat2,
		PI,
		mul,
		div,
		sub,
		add,
		atan,
		log,
		acos
	} from 'three/tsl';
	import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

	// Adaptive DPR
	let targetFPS = 45;
	let frameCount = 0;
	let elapsed = 0;
	function adjustDPR(renderer, delta) {
		elapsed += delta;
		frameCount++;

		if (elapsed >= 0.094) {
			let fps = frameCount * elapsed * 100;
			frameCount = 0;
			elapsed -= 0.1;

			if (fps < targetFPS * 0.95) {
				renderer.setPixelRatio(Math.min(1, Math.max(0.1, renderer.getPixelRatio() * 0.9)));
			} else if (fps > targetFPS) {
				renderer.setPixelRatio(Math.min(1, renderer.getPixelRatio() * 1.1));
			}

			//console.log(`DPR: ${renderer.getPixelRatio()} (FPS: ${fps})`)
		}
	}

	// Ambient Occluisuon
	class AmbientOcclusion {
		static options = {
			samples: 4,
			spread: 0.015,
			enabled: true,
			factor: 0.005
		};

		static samples = uniform(this.options.samples);
		static spread = uniform(this.options.spread);
		static enabled = uniform(int(this.options.enabled));
		static factor = uniform(this.options.factor);

		// @ts-ignore
		static render = Fn(([position, normal, scene]) => {
			const diffuse = float(1).toVar();
			If(this.enabled.equal(1), () => {
				const occlusion = float(0.0).toVar();

				Loop({ start: 0, end: this.samples, condition: '<=' }, ({ i }) => {
					const spacer = float(i).div(this.samples);
					const samplePos = position.add(normal.mul(spacer.mul(this.spread)));
					const distance = scene(samplePos);
					//occlusion.addAssign(smoothstep(0.0, 1, distance))
					occlusion.addAssign(smoothstep(0.0, this.factor, distance));
				});
				diffuse.assign(clamp(occlusion.div(this.samples), 0, 1.0));
			});
			return diffuse;
		});
	}

	// Atmospheric Scattering
	const atmosphericScattering = Fn(([position, direction]) => {
		const topColour = vec3(0.1, 0.2, 0.5).mul(2);
		const midColour = vec3(1.0, 0.4, 0.2);
		const bottomColour = vec3(0.0, 0.0, 0.133);

		const t = clamp(position.y.mul(0.5).add(0.5), 0.0, 1.0); // Horizon-based gradient
		const skyColour = mix(
			mix(bottomColour, vec3(0.75, 0.85, 0.95), t).mul(3),
			vec3(0),
			t.mul(1.25)
		);

		const radius = 0.005;
		const distance = length(position.sub(normalize(direction)));
		const sun = exp(distance.negate().mul(distance.div(radius)));
		skyColour.addAssign(vec3(1.0, 0.8, 0.5).mul(sun));

		// Mie scattering (orange glow near the sun)
		const mie = exp(distance.mul(3.0).pow(2.0).negate()).mul(0.5);
		const mieColor = vec3(midColour).mul(mie);
		skyColour.addAssign(mieColor.mul(1.5));

		// Rayleigh scattering (blue tint in the upper sky)
		const rayleigh = exp(position.y.mul(2.5)).mul(0.3);
		const rayleighColor = rayleigh.mul(topColour);
		skyColour.addAssign(rayleighColor);

		// Night Transition
		const nightFactor = smoothstep(-0.1, 0.1, direction.y);
		skyColour.mulAssign(nightFactor);

		return skyColour;
	});

	class ShadowMarcher {
		static options = {
			softness: 10,
			intensity: 0.15,
			maxSteps: 32,
			near: 0.01,
			far: 64,
			enabled: true,
			surfaceDistance: 0.001
		};

		static softness = uniform(this.options.softness);
		static intensity = uniform(this.options.intensity);
		static maxSteps = uniform(this.options.maxSteps);
		static near = uniform(this.options.near);
		static far = uniform(this.options.far);
		static enabled = uniform(int(this.options.enabled));
		static surfaceDistance = uniform(this.options.surfaceDistance);

		static render = Fn(([rayOrigin, rayDirection, scene]) => {
			const shadow = float(1).toVar();
			If(this.enabled.equal(1), () => {
				const accumulatedDistance = float(this.near).toVar();
				const distance = float(0).toVar();

				Loop({ start: 0, end: this.maxSteps, condition: '<' }, () => {
					distance.assign(scene(rayOrigin.add(rayDirection.mul(accumulatedDistance))));
					If(abs(distance.x).lessThan(this.surfaceDistance), () => {
						shadow.assign(0);
						Break();
					}).Else(() => {
						shadow.assign(min(shadow, this.softness.mul(distance.x).div(accumulatedDistance)));
						accumulatedDistance.addAssign(clamp(distance.x, this.surfaceDistance, 10));

						If(accumulatedDistance.greaterThan(this.far), () => {
							Break();
						});
					});
				});

				shadow.assign(max(this.intensity, shadow));
			});
			return shadow;
		});
	}

	// Helpers for a gentle tumble, spin steadily on Y and add a small wobble on X and Z
	// at different rates so it never repeats exactly.
	const rotX = (v, a) =>
		vec3(v.x, v.y.mul(cos(a)).sub(v.z.mul(sin(a))), v.y.mul(sin(a)).add(v.z.mul(cos(a))));

	const rotY = (v, a) =>
		vec3(v.x.mul(cos(a)).add(v.z.mul(sin(a))), v.y, v.z.mul(cos(a)).sub(v.x.mul(sin(a))));

	const rotZ = (v, a) =>
		vec3(v.x.mul(cos(a)).sub(v.y.mul(sin(a))), v.x.mul(sin(a)).add(v.y.mul(cos(a))), v.z);

	class MandelBulb {
		static options = {
			power: 8,
			iterations: 8
		};

		static power = uniform(this.options.power);
		static iterations = uniform(this.options.iterations);

		static speed = uniform(-0.1); // radians per second
		static wobble = uniform(0.25); // tumble amount (rad)

		static scene = Fn(([position]) => {
			// Rotate the sample point around Y
			const a = time.mul(this.speed);
			const c = cos(a);
			const s = sin(a);
			// just y:
			// const rotated = vec3(
			// 	position.x.mul(c).add(position.z.mul(s)),
			// 	position.y,
			// 	position.z.mul(c).sub(position.x.mul(s))
			// );

			// slow wobble:
			const rotated = rotZ(
				rotX(rotY(position, time.mul(this.speed)), sin(time.mul(0.4)).mul(this.wobble)),
				sin(time.mul(0.27)).mul(this.wobble)
			);
			// wobble sets how far it rocks: 0.25 is about 14°, and 0 gives you the plain Y spin back.
			// The two sin rates (0.4 and 0.27) don't share a common multiple, so the motion drifts rather than looping. For a fuller tumble, drop the sin and use time.mul(0.2) and time.mul(0.13) directly, which spins on all three axes continuously.

			const p = rotated.toVar();
			// const p = position.toVar(); static unrotating p

			const r = float(0).toVar();
			const dr = float(1).toVar();

			Loop({ start: 0, end: this.iterations }, () => {
				r.assign(length(p));

				If(r.greaterThan(2), () => {
					Break();
				});

				// Update the derivative accumulator based on radius
				const r_pow = r.pow(this.power.sub(1.0));
				dr.assign(r_pow.mul(this.power).mul(dr).add(1.0));

				// Convert to spherical coordinates
				const theta = acos(clamp(p.z.div(r), -1.0, 1.0));
				const phi = atan(p.y, p.x);

				// Scale by the fractal power
				const pr = pow(r, this.power);
				theta.mulAssign(this.power);
				phi.mulAssign(this.power);

				// Convert back to cartesian
				p.assign(pr.mul(vec3(sin(theta).mul(cos(phi)), sin(theta).mul(sin(phi)), cos(theta))));

				// Add to original position
				// p.addAssign(position);  static bulb non-rotating
				p.addAssign(rotated);
			});

			return log(r).mul(r).div(dr).mul(0.5);
		});

		static render = Fn(
			([baseColour, position, normal, rayDirection, lightPosition, lightDirection]) => {
				const diffuse = clamp(dot(normal, lightDirection), 0, 1).toVar();
				const shadow = ShadowMarcher.render(
					position.add(normal.mul(0.001)),
					lightDirection,
					this.scene
				);
				const colour = atmosphericScattering(
					reflect(rayDirection, normal),
					normalize(lightPosition)
				);
				colour.assign(mix(colour.mul(shadow), diffuse.mul(normal).mul(0.25).add(1), shadow));
				colour.mulAssign(AmbientOcclusion.render(position, normal, this.scene));
				// Gradient by distance from the centre: crevices -> tips
				const t = smoothstep(0.6, 1.2, length(position));

				const c1 = vec3(0.1, 0.2, 0.9); // deep
				const c2 = vec3(0.9, 0.2, 0.6); // mid
				const c3 = vec3(1.0, 0.8, 0.2); // tips
				// Fixed stops:
				// const grad = mix(mix(c1, c2, smoothstep(0.0, 0.5, t)), c3, smoothstep(0.5, 1.0, t));
				// For a smoothly cycling rainbow instead of fixed stops, use the following cosine palette://
				const grad = vec3(0.5).add(
					vec3(0.5).mul(
						cos(
							mul(PI, 2).mul(
								vec3(1.0, 1.0, 1.0)
									.mul(t.add(time.mul(0.05)))
									.add(vec3(0.0, 0.33, 0.67))
							)
						)
					)
				);
				// 0 = no tint, 1 = full gradient
				colour.assign(colour.mul(mix(vec3(1), grad, 0.85)));
				return colour;
			}
		);
	}

	class SDFScene {
		static options = {
			maxSteps: 256,
			surfaceDistance: 0.0001,
			cameraNear: 0.1,
			cameraFar: 100.0
		};

		static maxSteps = uniform(this.options.maxSteps);
		static surfaceDistance = uniform(this.options.surfaceDistance);
		static cameraNear = uniform(this.options.cameraNear);
		static cameraFar = uniform(this.options.cameraFar);

		static scene = Fn(([position]) => {
			return vec2(MandelBulb.scene(position), 0).toVar();
		});

		static getNormal = Fn(([position, distance]) => {
			const offset = vec2(0.0025, 0);

			return normalize(
				distance.sub(
					vec3(
						this.scene(position.sub(offset.xyy)).x,
						this.scene(position.sub(offset.yxy)).x,
						this.scene(position.sub(offset.yyx)).x
					)
				)
			);
		});

		static render = Fn(([rayOrigin_immutable]) => {
			const rayOrigin = rayOrigin_immutable.toVar();

			const p = positionLocal;

			const rayDirection = normalize(p).toVar();

			const t = time; //.div(5)
			//const lightPosition = vec3(0, 50, this.cameraFar.negate())
			//const lightPosition = vec3(0, sin(t).mul(30).add(43), this.cameraFar.negate())
			const lightPosition = vec3(sin(t).mul(this.cameraFar), 50, cos(t).mul(this.cameraFar));
			//const lightPosition = vec3(sin(t).mul(this.cameraFar), sin(t).mul(40).add(50), cos(t).mul(this.cameraFar))
			const lightDirection = normalize(lightPosition.sub(p)).toVar();

			const skyColour = atmosphericScattering(p, normalize(lightPosition)).toVar();
			const finalColour = skyColour.toVar();

			const accumulatedDistance = float(this.cameraNear).toVar();
			const fogDistance = accumulatedDistance.toVar();

			const distance = vec2(0).toVar();
			const position = vec3(0).toVar();

			Loop({ start: 0, end: this.maxSteps }, () => {
				position.assign(rayOrigin.add(rayDirection.mul(accumulatedDistance)));
				distance.assign(this.scene(position));

				If(
					abs(distance.x)
						.lessThan(this.surfaceDistance)
						.or(accumulatedDistance.greaterThan(this.cameraFar)),
					() => {
						Break();
					}
				);

				accumulatedDistance.addAssign(distance.x);
				fogDistance.addAssign(distance.x);
			});

			const normal = this.getNormal(position, distance.x).toVar();

			If(accumulatedDistance.lessThan(this.cameraFar), () => {
				finalColour.assign(
					MandelBulb.render(
						finalColour,
						position,
						normal,
						rayDirection,
						lightPosition,
						lightDirection
					)
				);
			});

			return finalColour;
		});
	}

	const scene = new THREE.Scene();

	const camera = new THREE.PerspectiveCamera(53, window.innerWidth / window.innerHeight, 0.1, 1000);
	camera.position.set(1, 1, 2.25);

	const renderer = new THREE.WebGPURenderer();
	renderer.setSize(window.innerWidth, window.innerHeight);
	document.body.appendChild(renderer.domElement);
	renderer.setAnimationLoop(animate);
	renderer.setPixelRatio(0.25); // start low for slower cards

	window.addEventListener('resize', function () {
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);
		controls.handleResize();
	});

	const controls = new OrbitControls(camera, renderer.domElement);
	controls.enableDamping = true;

	const camPos = uniform(new THREE.Vector3());

	scene.backgroundNode = SDFScene.render(camPos.toVar());

	const timer = new THREE.Timer();

	function animate() {
		adjustDPR(renderer, timer.update().getDelta()); //Adaptive DPR

		controls.update();

		camPos.value.copy(camera.position);

		renderer.render(scene, camera);
	}
</script>
